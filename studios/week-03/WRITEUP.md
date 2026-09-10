# Week 3 Studio — Modes and Misuse

**Control Scorecard — Task 4** · Evidencia: `python3 test_modes.py`

```
  ok  ECB leaks 2 blocks, CBC hides (96) — the MODE decided, not the cipher
  ok  CTR nonce reuse recovered m2 = week-2 two-time pad on a modern mode
  ok  H(secret‖msg) guarantee COLLAPSED — valid tag forged without the key
  ok  HMAC rejected the forgery — better construction, not a better hash
all 4 tests pass
```

---

## Scorecard

| Construcción | Garantía (eje 2), con su condición | Cómo falla | Clasificación |
|---|---|---|---|
| **ECB** | Confidencialidad del contenido de **cada bloque por separado**, y nada más. No promete ocultar la relación entre bloques. | Bloques iguales → ciphertext igual. Los 96 bloques de la imagen colapsan a **2 distintos**, los mismos 2 del plaintext: la estructura pasa 1:1. | **Misuse** |
| **CBC / CTR** | Confidencialidad del mensaje completo, **siempre y cuando** el par (nonce, contador) nunca se repita bajo la misma llave — y, en CBC, el IV sea impredecible. | Con nonce repetido el keystream se comparte y se cancela: `c1 ⊕ c2 = m1 ⊕ m2`. La llave sale de la ecuación. Recuperado íntegro: `the quarterly meeting moved to three pm`. CBC, dentro de su condición, dio 96/96 bloques distintos. | **Misuse** |
| **`H(secreto‖msg)`** | *Aparenta* autenticar: "solo quien tiene el secreto produce un tag válido". La garantía es **falsa** para cualquier hash Merkle–Damgård. | El digest **es** el estado interno, así que el atacante reanuda el hasheo desde el tag. Con solo `(msg, tag, len(secreto))` se forjó un tag para `amount=100&to=alice\x00\x00&to=attacker`; el servidor calculó el mismo `2591017062` y lo aceptó. | **Misuse** |
| **HMAC** | Autenticación de integridad y origen, **siempre que** la llave permanezca secreta. El anidamiento no expone estado reanudable. | No falló: rechazó la falsificación sin llave. | — |

Los tres fallos son **misuse**. Ninguno tocó AES ni SHA-256: la primitiva no es el
control; el modo y la construcción sí lo son.

---

## Nonce único, llave reutilizada

> *¿Sigue CTR siendo seguro? Enuncia la condición con precisión.*

Sí. Reutilizar la llave es el uso normal — la condición nunca fue sobre la llave:

> AES-CTR preserva la confidencialidad **siempre y cuando el par (nonce, contador)
> nunca se repita bajo la misma llave.**

Dos matices que "el nonce es único" no cubre: el contador de un mensaje muy largo
puede alcanzar el bloque de otro mensaje y repetir el par pese a nonces distintos;
y si los nonces se eligen al azar, lo que manda es la probabilidad de colisión, no
la unicidad nominal. Un millón de mensajes está holgado; el margen se agota mucho
antes de agotar el espacio de nonces.

---

## Dónde pude haber sido injusto, y qué no probé

- La primitiva no es AES: `block_cipher` es `sha256(key+block)` y **no es
  invertible**, así que no hay descifrado. No pude demostrar el *cut-and-paste*
  sobre ECB, su otro fallo grave.
- `md_hash` es un Merkle–Damgård de juguete con estado de 32 bits. El ataque de
  extensión transfiere verbatim a SHA-256; la fragilidad del hash de juguete no.
- Cobertura de una sola muestra: un par de mensajes y una falsificación, sobre las
  entradas fijas de `data.py`. El secreto me dieron su longitud; el ataque real
  exige iterar sobre longitudes candidatas.
- CBC salió indemne pero **no lo ataqué**: no probé IV fijo ni predecible, ni
  padding oracle. "CBC oculta la estructura" es lo único que medí.
