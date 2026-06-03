---
title: "c4ptur3-th3-fl4g"
date: 2026-01-01
draft: false
description: "Walkthrough de c4ptur3-th3-fl4g de TryHackMe. Traducción de encodings, espectrogramas y esteganografía — preparación eJPT."
tags:
  [
    "walkthrough",
    "tryhackme",
    "ejpt",
    "ctf",
    "encoding",
    "steganography",
    "cryptography",
    "base64",
    "binary",
    "morse",
  ]
categories: ["Walkthroughs"]
series: ["Road to eJPTv2"]
ShowToc: true
TocOpen: false
weight: 7
---

## Resumen

**c4ptur3-th3-fl4g** es la séptima máquina de la serie _Road to eJPTv2_ y la más diferente hasta ahora. No hay explotación de servicios, no hay reverse shells, no hay privesc. Es un reto de **codificación, criptografía y esteganografía** puro — diseñado para familiarizarte con los sistemas de representación de datos que aparecen constantemente en CTFs y en análisis forense.

Esta sala cubre: leetspeak, binario, Base32, Base64, hexadecimal, ROT13, ROT47, código Morse, BCD, Brainfuck/Malbolge, espectrogramas de audio y esteganografía de imágenes.

| Atributo       | Valor                                                         |
| -------------- | ------------------------------------------------------------- |
| **Plataforma** | TryHackMe                                                     |
| **Dificultad** | Fácil                                                         |
| **OS**         | N/A (reto de encoding)                                        |
| **Sala**       | [c4ptur3-th3-fl4g](https://tryhackme.com/room/c4ptur3th3fl4g) |
| **Skills**     | Encoding/Decoding, Steganography, Spectrograms, OSINT básico  |

### 🎥 Versión en video

{{< youtube yvhr_lWnCnQ >}}

> Si prefieres seguir el walkthrough paso a paso, continúa leyendo. El video cubre el mismo proceso en formato visual.

### Herramientas usadas

- [CyberChef](https://gchq.github.io/CyberChef/) — navaja suiza para encoding/decoding
- [Audacity](https://www.audacityteam.org/) — visualización de espectrogramas
- `steghide` — extracción de datos ocultos en imágenes
- `binwalk` — análisis de archivos compuestos

### Resumen de la solución

Esta sala se divide en cuatro secciones:

1. **Translations & Shifting** — 10 retos de encoding/decoding
2. **Spectrograms** — mensaje oculto en frecuencias de audio
3. **Steganography** — datos ocultos dentro de una imagen
4. **Security through obscurity** — archivo dentro de otro archivo

---

## Sección 1: Translations & Shifting

Esta sección presenta 10 cadenas codificadas que hay que decodificar. La herramienta principal es **CyberChef** (https://gchq.github.io/CyberChef/), que permite aplicar múltiples transformaciones en cadena.

### Reto 1: Leetspeak

`c4n y0u c4p7u23 7h3 f149?`

El **leetspeak** (l33tspeak) es una sustitución de letras por números o símbolos visualmente similares. Común en cultura hacker desde los años 80.

| Leetspeak | Letra |
| --------- | ----- |
| 4         | a     |
| 0         | o     |
| 7         | t     |
| 2         | r     |
| 3         | e     |
| 1         | i     |
| 9         | g     |

> **Respuesta:** `can you capture the flag?`

### Reto 2: Binario

```
01101100 01100101 01110100 01110011 00100000 01110100 01110010 01111001 00100000 01110011 01101111 01101101 01100101 00100000 01100010 01101001 01101110 01100001 01110010 01111001 00100000 01101111 01110101 01110100 00100001
```

Cada grupo de 8 bits representa un carácter ASCII. En CyberChef: `From Binary`.

> **Respuesta:** `lets try some binary out!`

### Reto 3: Base32

```
MJQXGZJTGIQGS4ZAON2XAZLSEBRW63LNN5XCA2LOEBBVIRRHOM======
```

Base32 usa el alfabeto A-Z y 2-7, con `=` como padding. En CyberChef: `From Base32`.

> **Respuesta:** `base32 is super common in CTF's`

### Reto 4: Base64

```
RWFjaCBCYXNlNjQgZGlnaXQgcmVwcmVzZW50cyBleGFjdGx5IDYgYml0cyBvZiBkYXRhLg==
```

Base64 usa A-Z, a-z, 0-9, +, / y `=` como padding. Reconocible por los `==` al final. En CyberChef: `From Base64`.

> **Respuesta:** `Each Base64 digit represents exactly 6 bits of data.`

### Reto 5: Hexadecimal

```
68 65 78 61 64 65 63 69 6d 61 6c 20 6f 72 20 62 61 73 65 31 36 3f
```

Cada par hex representa un byte. El `20` es el espacio en ASCII. En CyberChef: `From Hex`.

> **Respuesta:** `hexadecimal or base16?`

### Reto 6: ROT13

```
Ebgngr zr 13 cynprf!
```

ROT13 desplaza cada letra 13 posiciones en el alfabeto. Aplicado dos veces vuelve al original (es su propio inverso). En CyberChef: `ROT13`.

> **Respuesta:** `Rotate me 13 places!`

### Reto 7: ROT47

```
*@F DA:? >6 C:89E C@F?5 323J C:89E C@F?5 Wcf E:>6DX
```

ROT47 es como ROT13 pero aplica sobre 94 caracteres ASCII imprimibles (del 33 al 126), no solo letras. Por eso puede cifrar símbolos, números y letras. En CyberChef: `ROT47`.

> **Respuesta:** `You spin me right round baby right round (47 times)`

### Reto 8: Código Morse

```
. .-.. . -.-. --- -- -- ..- -. .. -.-. .- - .. --- -.
. -. -.-. --- -.. .. -. --.
```

El código Morse usa puntos y rayas para representar letras. En CyberChef: `From Morse Code`.

> **Respuesta:** `TELECOMMUNICATION ENCODING`

### Reto 9: BCD (Binary Coded Decimal)

```
85 110 112 97 99 107 32 116 104 105 115 32 66 67 68
```

BCD representa cada dígito decimal con su equivalente en bits. Estos valores son directamente códigos ASCII decimales. En CyberChef: `From Charcode` (base decimal).

> **Respuesta:** `Unpack this BCD`

### Reto 10: Cadena multi-capa

```
LS0tLS0gLi0tLS0g...
```

Esta cadena requiere decodificación en múltiples capas:

1. Base64 → produce código Morse
2. Morse → produce texto en claro

En CyberChef: `From Base64` → `From Morse Code`.

> **Respuesta:** `Let's make this a bit trickier...`

---

## Sección 2: Espectrogramas

Un **espectrograma** es una representación visual de las frecuencias de una señal de audio a lo largo del tiempo. Los mensajes se pueden ocultar en audio configurando el volumen de frecuencias específicas para que formen letras o imágenes cuando se visualizan en un espectrograma.

### Proceso

1. Descarga el archivo de audio de la sala
2. Ábrelo en **Audacity**
3. En la pista de audio, haz click en el nombre de la pista → selecciona **"Spectrogram"**
4. El espectrograma revelará texto visible en las frecuencias

> **Respuesta:** `Super Secret Message`

---

## Sección 3: Esteganografía

La **esteganografía** oculta información dentro de otros archivos — imágenes, audio, video — de forma que no sea visible a simple vista. A diferencia de la criptografía (que cifra el mensaje), la esteganografía oculta la existencia del mensaje.

### Proceso

1. Descarga la imagen de la sala
2. Usa `steghide` para extraer datos ocultos:

```bash
steghide extract -sf imagen.jpg
```

3. Si tiene passphrase, intenta con cadena vacía (Enter sin escribir nada)
4. El archivo extraído contiene la respuesta

> **Respuesta:** `SpaghettiSteg`

---

## Sección 4: Security through obscurity

Esta sección muestra cómo los atacantes (y defensores) pueden ocultar archivos dentro de otros archivos — una técnica usada tanto en malware como en CTFs.

### Reto 1: Archivo dentro de un archivo

1. Descarga el archivo de la sala
2. Usa `binwalk` para ver qué hay dentro:

```bash
binwalk archivo
```

3. Extrae el contenido:

```bash
binwalk -e archivo
```

4. Dentro encontrarás el archivo oculto

> **Primer archivo oculto:** `hackerchat.png`

### Reto 2: Texto oculto dentro del archivo

1. Con el archivo extraído, inspecciónalo:

```bash
strings hackerchat.png
```

O ábrelo con un editor hexadecimal y busca texto en claro al final del archivo.

> **Texto oculto:** `AHH_YOU_FOUND_ME!`

---

## Referencia rápida de encodings

Esta tabla resume los encodings más comunes en CTFs:

| Encoding    | Característica visual           | Herramienta            |
| ----------- | ------------------------------- | ---------------------- |
| Binario     | Solo 0s y 1s en grupos de 8     | CyberChef: From Binary |
| Hexadecimal | Caracteres 0-9 y a-f en pares   | CyberChef: From Hex    |
| Base32      | Mayúsculas + 2-7, padding con = | CyberChef: From Base32 |
| Base64      | A-Z, a-z, 0-9, +/, padding ==   | CyberChef: From Base64 |
| ROT13       | Texto legible pero desplazado   | CyberChef: ROT13       |
| ROT47       | Símbolos mezclados con texto    | CyberChef: ROT47       |
| Morse       | Puntos, rayas y espacios        | CyberChef: From Morse  |
| Leetspeak   | Números mezclados con letras    | Inspección manual      |

---

## Lecciones aprendidas

- **CyberChef es indispensable para CTFs** — La mayoría de los encodings de esta sala se resuelven en segundos con CyberChef. Aprender a encadenar operaciones en CyberChef es una habilidad fundamental para cualquier CTF player.
- **Reconocer encodings a primera vista ahorra tiempo** — Los `==` al final → Base64. Las mayúsculas con 2-7 → Base32. Puntos y rayas → Morse. Los 0s y 1s en grupos de 8 → Binario. Construir este reconocimiento visual es esencial.
- **La esteganografía no es solo imágenes** — Se puede ocultar información en audio (espectrogramas), video, documentos PDF, archivos ZIP, y más. Siempre que tengas un archivo en un CTF, pregúntate: ¿hay algo escondido aquí?
- **`binwalk` y `strings` son tus primeros comandos ante archivos desconocidos** — `strings` muestra texto legible dentro de cualquier archivo binario. `binwalk` detecta archivos embebidos. Úsalos antes de abrir cualquier archivo sospechoso.
- **Security through obscurity no es seguridad real** — Ocultar archivos dentro de otros o cambiar extensiones no protege los datos. Un atacante con `binwalk` o `strings` lo descubre en segundos. La seguridad real requiere cifrado, no ocultamiento.

### Para la eJPT

Aunque la eJPT se enfoca más en explotación de redes y sistemas, los conceptos de esta sala son relevantes para:

- Reconocer datos codificados en respuestas de aplicaciones web (Base64 en cookies, tokens JWT, etc.)
- Análisis forense básico en post-explotación
- Comprensión de cómo se oculta información en archivos

**Tiempo aproximado de resolución:** 30-45 minutos con CyberChef abierto y la referencia de encodings a la mano.

---

## Referencias

- [c4ptur3-th3-fl4g — TryHackMe](https://tryhackme.com/room/c4ptur3th3fl4g)
- [CyberChef — GCHQ](https://gchq.github.io/CyberChef/)
- [Audacity](https://www.audacityteam.org/)
- [steghide documentation](http://steghide.sourceforge.net/)
- [binwalk documentation](https://github.com/ReFirmLabs/binwalk)
- [GTFOBins](https://gtfobins.github.io/)
