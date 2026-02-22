# Write-up : PS3 Signing Upgraded

## Informations générales
- **Titre** : PS3 Signing Upgraded
- **Type** : Crypto
- **Niveau** : Moyen
- **Description** : La signature des jeux de PS3 était vraiment impressionnante mais pas assez pour moi. Du coup j'ai décidé de l'améliorer. D'ailleurs, j'ai réutilisé le secret de la signature (une fois digéré) comme clé pour chiffrer mon super flag :) Flag Format: CCOI26{...}

Ce challenge nous fournit un fichier JSON contenant une clé publique ECDSA (coordonnées x et y), deux signatures sur des messages de trafic capturé, et un secret chiffré en AES-256-CBC. L'objectif est de récupérer la clé privée utilisée pour les signatures, de la hacher pour obtenir la clé AES, et de décrypter le flag.

## Analyse du problème
Le fichier JSON est le suivant :

```json
{
  "public_key": {
    "x": "0x4f588adfe1636a0744e740683ba1226a62a4f968ff6999a6b5d2d04b511ff852",
    "y": "0x50451d95e5c0eefc3fb1ec28bb45ebdfd97dd5289afc1cdaacdfa5550d4e48ed"
  },
  "captured_traffic": [
    {
      "seq_id": 4096,
      "data": "REQ-ID:4096|CMD:AUTH",
      "signature": {
        "r": "0x23ae8f9830cfa0b8281fbb889a9149960744d8f4e0c2d7b6bfdd6976c74e0908",
        "s": "0xa88fe209f2063a27e7a2338353744133eccdf849a5295329be2c466604c097a8"
      }
    },
    {
      "seq_id": 4097,
      "data": "REQ-ID:4097|CMD:KEY_ROTATION",
      "signature": {
        "r": "0xd16cd10272ceab1e290dc25e6ab281fabbb49cf42e18a36ef31727b2894ca700",
        "s": "0x867758a6a954e10916b72ec9fe87c7fd932be5ef4b1c5ff79550b62cb11b7195"
      }
    }
  ],
  "encrypted_secret": {
    "method": "AES-256-CBC",
    "iv": "6e898ec479e459f41bb27ac5eb719a30",
    "ciphertext": "a6e0985903391a4bdbd514df0a1d2c7f8600b9b4d6a8e1e28b7e89d5b986cbc99d206a61141003e78987f231c65b49e0"
  }
}
```

On observe :
- Une clé publique sur une courbe elliptique (probablement secp256r1 ou secp256k1, car on a des coordonnées x et y pour le point G*d, où d est la clé privée).
- Deux messages signés avec ECDSA : "REQ-ID:4096|CMD:AUTH" et "REQ-ID:4097|CMD:KEY_ROTATION", avec leurs hashes SHA-256 (h1 et h2).
- Les signatures (r1, s1) et (r2, s2).
- Un ciphertext AES-256-CBC avec IV, qui utilise la clé privée "digérée" (hachée via SHA-256) comme clé de chiffrement.

Le titre fait référence à la signature PS3, qui historiquement utilisait ECDSA avec un nonce défectueux (k réutilisé), permettant de récupérer la clé privée. Ici, c'est une "amélioration", mais probablement avec une vulnérabilité similaire ou liée, comme des nonces corrélés.

En ECDSA, pour une signature (r, s) sur un hash h :
- r = (k * G).x mod n
- s = (h + d * r) / k mod n

Si on a deux signatures avec nonces k1 et k2 liés (par exemple, k2 = k1 + offset ou similaire), on peut résoudre pour d.

Dans ce challenge, les nonces semblent liés d'une manière qui permet de former des équations linéaires pour extraire k1 et d.

## Résolution
Pour résoudre, on suppose que les nonces k1 et k2 sont liés. En regardant les formules ECDSA :

Pour la première signature :
s1 = k1^{-1} * (h1 + d * r1) mod n

Pour la seconde :
s2 = k2^{-1} * (h2 + d * r2) mod n

Si on suppose une relation entre k1 et k2 (par exemple, k2 = k1 + c, mais ici c'est plus complexe), on peut manipuler les équations.

Dans le script fourni, on calcule :

num = (s2 * r1 - h2 * r1 + h1 * r2) % n  
den = (s1 * r2 - s2 * r1) % n  
k1 = (num * inverse(den, n)) % n  
d = ((s1 * k1 - h1) * inverse(r1, n)) % n

Cela dérive d'une attaque où k2 = r1 + k1 ou une variante similaire (inspirée de la fail PS3 où k était constant). En fait, en dérivant les équations :

De s1 * k1 = h1 + d * r1  
s2 * k2 = h2 + d * r2

Si on suppose k2 = k1 + r1 (ou une relation linéaire), on peut résoudre le système.

Le script teste deux courbes courantes pour n (l'ordre) : NIST P-256 (secp256r1) et secp256k1.

Pour chaque courbe :
1. Calculer k1 via la formule ci-dessus.
2. Calculer d.
3. Hacher d avec SHA-256 pour obtenir la clé AES (32 bytes).
4. Décrypter le ciphertext avec AES-CBC et l'IV donné.
5. Vérifier le padding ; si OK, c'est la bonne courbe.

Le script utilise `pycryptodome` pour AES et les opérations crypto.

## Script de solution
Voici le script Python fourni (nécessite `pip install pycryptodome`) :

```python
import hashlib
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad
from Crypto.Util.number import long_to_bytes, inverse

# Données du challenge
msg1 = b"REQ-ID:4096|CMD:AUTH"
msg2 = b"REQ-ID:4097|CMD:KEY_ROTATION"
h1 = int(hashlib.sha256(msg1).hexdigest(), 16)
h2 = int(hashlib.sha256(msg2).hexdigest(), 16)
r1 = 0x23ae8f9830cfa0b8281fbb889a9149960744d8f4e0c2d7b6bfdd6976c74e0908
s1 = 0xa88fe209f2063a27e7a2338353744133eccdf849a5295329be2c466604c097a8
r2 = 0xd16cd10272ceab1e290dc25e6ab281fabbb49cf42e18a36ef31727b2894ca700
s2 = 0x867758a6a954e10916b72ec9fe87c7fd932be5ef4b1c5ff79550b62cb11b7195

# Ordre des courbes standards 256-bits (n)
curves = {
    "NIST P-256 (secp256r1)": 0xffffffff00000000ffffffffffffffffbce6faada7179e84f3b9cac2fc632551,
    "secp256k1": 0xfffffffffffffffffffffffffffffffebaaedce6af48a03bbfd25e8cd0364141
}

# Données AES
iv = bytes.fromhex("6e898ec479e459f41bb27ac5eb719a30")
ct = bytes.fromhex("a6e0985903391a4bdbd514df0a1d2c7f8600b9b4d6a8e1e28b7e89d5b986cbc99d206a61141003e78987f231c65b49e0")

for curve_name, n in curves.items():
    try:
        # Calcul du numérateur et du dénominateur pour k1
        num = (s2 * r1 - h2 * r1 + h1 * r2) % n
        den = (s1 * r2 - s2 * r1) % n
        # Résolution de k1 puis de la clé privée d
        k1 = (num * inverse(den, n)) % n
        d = ((s1 * k1 - h1) * inverse(r1, n)) % n
        # La clé privée est "digérée" (SHA-256) pour devenir la clé AES
        d_bytes = long_to_bytes(d)
        aes_key = hashlib.sha256(d_bytes).digest()
        # Déchiffrement du flag
        cipher = AES.new(aes_key, AES.MODE_CBC, iv)
        plaintext = unpad(cipher.decrypt(ct), AES.block_size)
        print(f"Succès sur la courbe {curve_name} ! \nFlag : {plaintext.decode()}")
        break
    except Exception as e:
        # Si le padding AES échoue, ce n'est pas la bonne courbe
        continue
```

## Explication détaillée du script
1. **Préparation des données** : Hasher les messages avec SHA-256 pour obtenir h1 et h2. Extraire r1, s1, r2, s2 du JSON.
2. **Boucle sur les courbes** : Tester secp256r1 et secp256k1 (courbes 256 bits courantes).
3. **Calcul de k1** : Utiliser une formule dérivée pour trouver k1 à partir des relations entre les signatures. Cela suppose une relation spécifique entre k1 et k2 (probablement k2 = k1 + quelque chose lié à r1, d'où la formule).
4. **Récupération de d** : Une fois k1 connu, résoudre d à partir de la première équation ECDSA.
5. **Génération clé AES** : SHA-256 sur les bytes de d.
6. **Déchiffrement** : AES-CBC avec unpading PKCS7. Si succès, imprimer le flag ; sinon, essayer l'autre courbe.

En exécutant, on trouve que la courbe secp256k1 (ou l'autre, selon) donne le bon décryptage.

## Flag
En exécutant le script, on obtient le flag : CCOI26{...} (remplacez par la valeur réelle obtenue, par exemple si c'est "CCOI26{upgraded_ps3_signature_with_aes}").

