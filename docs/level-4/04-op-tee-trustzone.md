# 04 · Trusted Execution — OP-TEE & TrustZone

Secure boot (Module 2) protects what runs at startup. TrustZone protects
something different: a region of the *running* system — key material, a
PIN comparison, a DRM decision — that must stay isolated from Linux
itself, so that a full kernel exploit still doesn't get the keys. OP-TEE
is the open-source Trusted OS that runs in ARM's TrustZone Secure World
alongside Linux running in the Normal World.

## The two-world model

```
┌─────────────────────────────┐   ┌─────────────────────────────┐
│  Normal World (Non-secure)   │   │  Secure World                │
│  Linux kernel + userspace    │   │  OP-TEE (Trusted OS)          │
│                               │   │                               │
│  tee-supplicant (daemon) ────┼──▶│  Trusted Applications (TAs)   │
│  libteec (client API)  ◀─────┼───┤  run here, isolated memory    │
└──────────────┬────────────────┘   └───────────────┬──────────────┘
               │         SMC (Secure Monitor Call)    │
               └────────────────────────────────────────┘
                    Arbitrated by ARM Trusted Firmware (BL31)
```

Crucially, Linux **cannot read Secure World memory** — this isn't an
access-control list Linux chooses to respect, it's enforced by the
memory controller/TZASC at a level below the kernel entirely. A root
exploit in Linux does not, by itself, grant access to whatever OP-TEE is
protecting; that isolation boundary is the entire value proposition.

## Client and Trusted Application: the two halves of any OP-TEE feature

A feature has a normal-world client (calls into OP-TEE) and a secure-world
TA (does the actual protected work). Client side, using `libteec`:

```c
#include <tee_client_api.h>

TEEC_Context ctx;
TEEC_Session sess;
TEEC_Result res;
uint32_t err_origin;

res = TEEC_InitializeContext(NULL, &ctx);
res = TEEC_OpenSession(&ctx, &sess, &ta_uuid,
			TEEC_LOGIN_PUBLIC, NULL, NULL, &err_origin);

TEEC_Operation op = { 0 };
op.paramTypes = TEEC_PARAM_TYPES(TEEC_MEMREF_TEMP_INPUT,
				  TEEC_MEMREF_TEMP_OUTPUT,
				  TEEC_NONE, TEEC_NONE);
op.params[0].tmpref.buffer = plaintext;
op.params[0].tmpref.size   = plaintext_len;
op.params[1].tmpref.buffer = ciphertext;
op.params[1].tmpref.size   = sizeof(ciphertext);

res = TEEC_InvokeCommand(&sess, TA_CMD_ENCRYPT, &op, &err_origin);

TEEC_CloseSession(&sess);
TEEC_FinalizeContext(&ctx);
```

TA side — this code physically executes inside Secure World, with its own
tiny libc and no access to Linux syscalls at all:

```c
#include <tee_internal_api.h>

TEE_Result TA_InvokeCommandEntryPoint(void *session, uint32_t cmd,
				       uint32_t param_types,
				       TEE_Param params[4])
{
	switch (cmd) {
	case TA_CMD_ENCRYPT: {
		TEE_ObjectHandle key_handle;
		TEE_OperationHandle op_handle;

		/* Key never leaves Secure World, ever. */
		TEE_OpenPersistentObject(TEE_STORAGE_PRIVATE, "device_key",
					  10, TEE_DATA_FLAG_ACCESS_READ,
					  &key_handle);
		TEE_AllocateOperation(&op_handle, TEE_ALG_AES_GCM,
				       TEE_MODE_ENCRYPT, 256);
		TEE_SetOperationKey(op_handle, key_handle);
		TEE_CipherDoFinal(op_handle,
				   params[0].memref.buffer, params[0].memref.size,
				   params[1].memref.buffer, &params[1].memref.size);
		return TEE_SUCCESS;
	}
	default:
		return TEE_ERROR_NOT_SUPPORTED;
	}
}
```

The design principle in that TA: the encryption key is opened as a
**persistent object inside Secure World storage** and never crosses the
SMC boundary in either direction — only plaintext in, ciphertext out. A
TA that instead reads the key and returns it to the client for the client
to do the crypto itself has completely defeated the point of using
TrustZone at all.

## tee-supplicant: the piece people forget

OP-TEE's kernel driver handles the SMC transport, but persistent secure
storage, RPC-based file access, and some crypto operations are serviced
by a **normal-world userspace daemon**, `tee-supplicant`. Without it
running, TA calls that need storage or RPC services hang or fail:

```console
$ systemctl status tee-supplicant
● tee-supplicant.service
   Active: active (running)
$ ls /dev/tee*
/dev/tee0  /dev/teepriv0
```

A very common "OP-TEE integration works in my quick smoke test but fails
in the actual product image" bug is `tee-supplicant` not being started
early enough (or at all) in the init sequence — any TA using persistent
storage before it's up will fail in a way that looks like a TA bug but is
actually a service ordering bug in systemd unit dependencies.

## Building a TA and getting the DT/boot wiring right

```console
$ export TA_DEV_KIT_DIR=/path/to/optee_os/out/arm/export-ta_arm64
$ make -C ta/ CROSS_COMPILE=aarch64-linux-gnu- \
    TA_DEV_KIT_DIR=$TA_DEV_KIT_DIR
$ ls ta/*.ta
8aaaf200-2450-11e4-abe2-0002a5d5c51b.ta
```

The `.ta` file's name **is** its UUID — this is not cosmetic, it's how
OP-TEE's TA loader finds the right binary when a client requests that
UUID. It's deployed to a location `tee-supplicant` scans (commonly
`/lib/optee_armtz/`), and a mismatch between the UUID compiled into the
client and the actual filename is a very literal, very common source of
"TA not found" failures that have nothing to do with the TA's actual code.

```dts
firmware {
	optee {
		compatible = "linaro,optee-tz";
		method = "smc";
	};
};
```

OP-TEE itself is loaded by the earlier boot stages (BL2/BL31 in ARM
Trusted Firmware, or U-Boot's `bootm` with a signed OP-TEE image), and the
DT node above only tells Linux's OP-TEE driver that Secure World support
exists at all — it does not itself install OP-TEE, which is a separate,
earlier bring-up step entirely and one of the places this integration
most commonly goes wrong on a new board port.

## Traps

- **Returning key material across the SMC boundary "just for
  convenience"** — see the TA example above; this collapses the entire
  security boundary the moment it happens even once, and it's very easy
  to accidentally do while debugging ("let me just print the key from
  Linux to check it") and then forget to remove.
- **`tee-supplicant` not running or started too late** — see above; the
  failure mode looks exactly like a broken TA.
- **Reusing a TA UUID across genuinely different TAs** (copy-pasted
  boilerplate without regenerating the UUID) — whichever `.ta` file is
  present under that UUID on a given build wins, silently, with no
  warning that a naming collision occurred.
- **Assuming OP-TEE's presence alone secures a feature.** OP-TEE only
  protects what a *correctly designed* TA protects — a TA that does trivial
  work while the actual sensitive logic stays in Linux userspace gains
  nothing from the TrustZone boundary it nominally uses.

## Cheat sheet

| Component | Runs where | Role |
|---|---|---|
| `libteec` (client API) | Normal World, userspace | Opens sessions, invokes TA commands |
| Trusted Application (`.ta`, named by UUID) | Secure World | Does the actual protected work |
| `tee-supplicant` | Normal World, userspace daemon | Services storage/RPC requests from TAs |
| OP-TEE driver | Normal World kernel | SMC transport between Linux and Secure World |
| ARM Trusted Firmware (BL31) | EL3, both worlds | Arbitrates the SMC world switch |

!!! note "On verification"
    The client/TA API usage and two-world architecture are checked
    against OP-TEE's documented `libteec`/`tee_internal_api` and the
    ARM TrustZone SMC model; no TA was built, signed, or invoked against
    real or emulated Secure World hardware from this machine.

## Exercise

(1) Sketch (client + TA pseudocode, following the pattern above) a "PIN
verify" TA that takes a PIN attempt from Linux, compares it against a
value stored only in Secure World persistent storage, and returns only a
boolean — explain specifically what must never cross the SMC boundary in
either direction for this design to be meaningful. (2) A teammate proposes
skipping `tee-supplicant` "since our TA doesn't need storage" — identify
which of your design's operations (if any) still implicitly need it. (3)
One paragraph: explain, to someone who thinks "OP-TEE" and "TrustZone" are
interchangeable, the actual relationship between the two.
