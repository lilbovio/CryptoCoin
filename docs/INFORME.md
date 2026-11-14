<!--
BORRADOR DEL INFORME TÉCNICO — formato de la plantilla del profesor (estilo paper).
Longitud objetivo: 4–6 páginas al exportar a PDF. Una columna, Times New Roman, encabezados numerados en negrita.
Antes de entregar:
  1. Reemplaza todo lo marcado como [COMPLETAR], [AUTOR], [CAPTURA: ...] y [AJUSTAR].
  2. Actualiza la sección 3 con la arquitectura final (docs/ARQUITECTURA.md) y la sección 4 con resultados reales.
  3. Verifica cada referencia de la sección References.
  4. Borra este comentario.
-->

# CryptoCoin: una base de datos encadenada con hash (mini-blockchain) para detectar alteraciones

**[AUTOR]**¹

¹ Universidad de Guadalajara, Centro Universitario de Tonalá. Ingeniería en Ciencias Computacionales. [CORREO]

**Resumen.** Este trabajo presenta CryptoCoin, una aplicación sencilla que registra transferencias de una criptomoneda básica en una base de datos donde cada registro incluye un hash SHA-256 que depende del registro anterior. De esta forma se simula, de manera rudimentaria, la idea central de una blockchain: modificar cualquier registro pasado rompe la cadena de hashes. La aplicación permite agregar registros, mostrar la cadena, verificar su integridad e identificar todos los registros alterados, distinguiendo entre contenido modificado y enlace roto. Los datos persisten en SQLite y se incluye un simulador de ataque para demostrar la detección. El sistema no pretende ser una blockchain completa: carece de consenso distribuido, minería, red P2P y firmas digitales.

**Palabras clave:** Criptografía, Función hash, SHA-256, Blockchain, Integridad de datos, Cadena de hashes.

## 1 Introducción

Una función hash criptográfica toma una entrada de longitud arbitraria y produce una salida de longitud fija, llamada resumen o *digest*. SHA-256, definida en el estándar FIPS 180-4 [1], produce resúmenes de 256 bits y tiene tres propiedades que se aprovechan en este proyecto: es **determinista** (la misma entrada siempre produce el mismo resumen), presenta **efecto avalancha** (un cambio mínimo en la entrada altera el resumen de forma impredecible) y es **resistente a preimagen y a colisiones** (en la práctica no es factible encontrar una entrada que produzca un resumen dado, ni dos entradas con el mismo resumen).

El efecto avalancha se puede observar con un ejemplo real: los resúmenes SHA-256 de las cadenas `CryptoCoin` y `cryptocoin`, que solo difieren en el uso de mayúsculas, son respectivamente `32c14024…1692` y `3daeb429…0fbc`, y difieren en 132 de sus 256 bits.

Una blockchain es un registro de datos formado por bloques enlazados criptográficamente: cada bloque incluye el hash del bloque anterior, de modo que alterar un bloque pasado invalida todos los enlaces posteriores [2][3]. Este proyecto simula únicamente esa idea de **encadenamiento**. No se busca implementar una blockchain completa, sino demostrar tres cosas:

1. El cálculo de hashes criptográficos sobre registros de datos.
2. El encadenamiento entre registros mediante el hash del registro anterior.
3. La detección de alteraciones a partir de la verificación de la cadena.

El tema elegido es una criptomoneda básica, CryptoCoin, cuyos registros son transferencias entre cuentas. Este tema es solo un ejemplo de datos; el mecanismo de encadenamiento es independiente de él.

## 2 Estado del arte

La idea de encadenar valores mediante funciones hash tiene una larga historia. Lamport propuso cadenas de hashes para autenticación de contraseñas en 1981 [4]. Haber y Stornetta (1991) las aplicaron al sellado de tiempo de documentos digitales, enlazando cada documento con el anterior de forma que no se pudiera alterar el orden ni el contenido sin ser detectado [5]. Nakamoto (2008) combinó el encadenamiento de bloques con prueba de trabajo y una red entre pares para construir Bitcoin, un sistema de dinero electrónico sin autoridad central [2]. Una exposición didáctica de estos conceptos se encuentra en Narayanan et al. [3].

CryptoCoin retoma únicamente el primer ingrediente, el encadenamiento por hash. Los demás elementos de una blockchain real (consenso, minería y red P2P) quedan fuera del alcance, como se discute en la sección 5.

## 3 Metodología utilizada

### 3.1 Estructura de datos

Cada registro, o bloque, contiene los campos de la Tabla 1.

| Campo | Tipo | Descripción |
|---|---|---|
| `id` | entero | Número de bloque. El génesis es 0 y cada bloque nuevo suma 1 |
| `timestamp` | cadena | Fecha y hora UTC, formato `AAAA-MM-DDTHH:MM:SS.ffffffZ` |
| `data` | objeto | Transacción `{from, to, amount}` con monto entero positivo |
| `prev_hash` | cadena hex (64) | Hash del bloque anterior; 64 ceros en el génesis |
| `hash` | cadena hex (64) | Hash propio del bloque |

**Tabla 1.** Campos de un bloque.

### 3.2 Lógica de encadenamiento

El hash de un bloque *i* se calcula a partir de su identificador, su marca de tiempo, sus datos y el hash del bloque anterior:

$$h_i = \text{SHA-256}(i \;\|\; t_i \;\|\; d_i \;\|\; h_{i-1}) \tag{1}$$

donde $\|$ denota concatenación. En la implementación, la concatenación se hace sobre la cadena `"{id}|{timestamp}|{data}|{prev_hash}"` codificada en UTF-8, y `data` se serializa en JSON canónico (claves ordenadas, sin espacios). Esta serialización fija es indispensable: si el mismo registro se serializara de dos formas distintas, se obtendrían hashes distintos y la verificación fallaría sin que hubiera alteración alguna. El carácter `|` no genera ambigüedad porque `id`, `timestamp` y `prev_hash` tienen formatos fijos que nunca lo contienen.

El primer bloque, llamado **génesis**, tiene `id = 0`, `prev_hash` igual a 64 ceros y una transacción nula. Se crea automáticamente cuando la base de datos está vacía. Cada bloque posterior toma como `prev_hash` el `hash` del último bloque almacenado, formando la cadena.

La Tabla 2 muestra una cadena de ejemplo de tres bloques (hashes abreviados), calculada con la fórmula (1).

| id | data | prev_hash | hash |
|---|---|---|---|
| 0 | GENESIS → GENESIS, 0 | `0000…0000` | `d8b7ce7a…b412` |
| 1 | SYSTEM → alice, 100 | `d8b7ce7a…b412` | `ef4ac89c…34ef` |
| 2 | alice → bob, 30 | `ef4ac89c…34ef` | `eb8c3c63…2c00` |

**Tabla 2.** Ejemplo de cadena de tres bloques.

### 3.3 Reglas de las transacciones

Antes de crear un bloque se valida la transacción: el monto debe ser un entero positivo, el emisor y el receptor deben ser distintos y no vacíos, y el emisor debe tener saldo suficiente. El emisor especial `SYSTEM` puede emitir monedas nuevas sin restricción de saldo; esto permite que haya monedas en circulación sin implementar minería. Una transacción inválida se rechaza y no genera ningún bloque.

### 3.4 Algoritmo de verificación

La verificación recorre **todos** los bloques en orden y no se detiene en el primer error. Para cada bloque se comprueba:

```
para cada bloque b en la cadena, en orden:
    si b es el génesis y no cumple la definición:      reportar GENESIS_INVALID
    si b.id != id_anterior + 1:                        reportar ID_GAP
    si faltan campos o su formato es incorrecto:       reportar MALFORMED
    si SHA-256(serialización(b)) != b.hash:            reportar HASH_MISMATCH
    si b.prev_hash != hash_almacenado(bloque anterior): reportar LINK_BROKEN
reproducir saldos con los datos almacenados:
    si algún emisor gasta más de lo que tiene:         reportar INSUFFICIENT_BALANCE
```

La distinción entre `HASH_MISMATCH` y `LINK_BROKEN` es importante. Si un atacante modifica el contenido de un bloque sin tocar su hash, el hash recalculado no coincide con el almacenado (`HASH_MISMATCH`). Si además recalcula y reescribe ese hash, el bloque parece consistente, pero el bloque siguiente conserva el `prev_hash` original y el enlace queda roto (`LINK_BROKEN`). El resultado se reporta como cadena válida o como lista de hallazgos con el `id` del bloque, el código de error y los valores esperado y encontrado.

### 3.5 Persistencia y arquitectura

Los bloques se almacenan en una base de datos SQLite, de modo que al cerrar y volver a abrir la aplicación los registros permanecen y la cadena puede verificarse. El acceso a los datos se realiza a través de una interfaz de almacenamiento independiente de la lógica criptográfica, lo que permitiría sustituir SQLite por otro respaldo sin modificar el núcleo. La aplicación está escrita en Python y utiliza únicamente su biblioteca estándar. **[COMPLETAR: resumir aquí la arquitectura definitiva (capas, módulos y diagrama) tomada de `docs/ARQUITECTURA.md`.]**

### 3.6 Simulador de ataque

Para demostrar la detección, la aplicación incluye una opción que modifica un registro pasado directamente en la base de datos, sin pasar por la lógica de la aplicación, tal como lo haría un atacante con acceso al archivo. Opcionalmente puede reescribir también el hash del bloque alterado para mostrar el caso del enlace roto. Después se ejecuta la verificación para observar el resultado.

## 4 Resultados experimentales

### 4.1 Agregar registros

Se agregaron transacciones a la cadena a través de la interfaz de la aplicación. **[CAPTURA: agregar al menos tres registros, incluyendo uno que emite monedas desde SYSTEM.]**

**Fig. 1.** Agregando registros a la cadena. [COMPLETAR]

### 4.2 Mostrar la cadena

**[CAPTURA: listado completo de la cadena con ids, marcas de tiempo, datos y hashes.]**

**Fig. 2.** Visualización de la cadena. [COMPLETAR]

### 4.3 Verificación de integridad

Con la cadena sin alteraciones, la verificación debe reportar una cadena válida. **[CAPTURA: salida de la verificación sobre una cadena íntegra.]**

**Fig. 3.** Verificación de una cadena íntegra. [COMPLETAR]

### 4.4 Detección de una alteración

Se simularon los escenarios de la Tabla 3 sobre la cadena de ejemplo. La columna de resultado esperado se obtiene del diseño; la columna de resultado observado debe completarse con la ejecución real.

| Escenario | Resultado esperado | Resultado observado |
|---|---|---|
| Ninguna alteración | Cadena válida | [COMPLETAR] |
| Bloque 1: monto 100 → 1000, hash sin cambio | `HASH_MISMATCH` en el bloque 1 | [COMPLETAR] |
| Igual al anterior, pero con el hash del bloque 1 reescrito | `LINK_BROKEN` en el bloque 2 | [COMPLETAR] |
| Bloque 2: monto 30 → 300, hash sin cambio | `HASH_MISMATCH` e `INSUFFICIENT_BALANCE` en el bloque 2 | [COMPLETAR] |
| Eliminar un bloque intermedio | `ID_GAP` y `LINK_BROKEN` | [COMPLETAR] |
| Alterar varios bloques | Todos se reportan, no solo el primero | [COMPLETAR] |

**Tabla 3.** Escenarios de alteración y detección.

**[CAPTURA: salida de la verificación tras ejecutar el simulador de ataque.]**

**Fig. 4.** Detección de una alteración. [COMPLETAR]

### 4.5 Pruebas automatizadas

Se desarrolló un conjunto de pruebas automatizadas que incluye los vectores de referencia de la Tabla 2, el caso de cadena válida, las alteraciones de cada campo, la eliminación de bloques, la persistencia tras reabrir la base de datos y el rechazo de transacciones inválidas. Resultado: **[COMPLETAR: número de pruebas ejecutadas y aprobadas]**.

## 5 Conclusiones y líneas de trabajo futuro

CryptoCoin demuestra que encadenar cada registro con el hash del anterior permite detectar la modificación de cualquier registro pasado, e identificar con precisión qué bloques fueron alterados y de qué forma. El sistema cumple el objetivo didáctico de ilustrar el cálculo de hashes, el encadenamiento y la verificación de integridad.

**Limitaciones.** Este sistema no es una blockchain completa:

- **No hay consenso distribuido.** Hay una sola copia de los datos y una sola parte que la controla.
- **No hay minería ni prueba de trabajo.** Recalcular todos los hashes tiene un costo despreciable.
- **No hay red P2P.** No existen otras copias contra las cuales comparar.
- **No hay firmas digitales.** Nada prueba quién creó un registro.
- **Un atacante con control total del archivo puede reescribir consistentemente todos los hashes posteriores** a un bloque alterado, y la cadena resultante parecería válida, porque no existe un ancla externa. La detección es confiable frente a alteraciones parciales, no frente a una reescritura completa de la cadena.
- Los saldos se calculan reproduciendo la cadena, lo cual no escala a historiales grandes.

**Mejoras posibles.**

- Firmar cada transacción con firma digital (por ejemplo, ECDSA o Ed25519) para autenticar al emisor.
- Publicar periódicamente el hash del último bloque en un medio externo como ancla de confianza.
- Replicar la cadena en múltiples nodos con un mecanismo de consenso.
- Agregar prueba de trabajo para encarecer la reescritura de la cadena.
- Usar un árbol de Merkle para verificar de forma eficiente la pertenencia de un registro.
- Sustituir SQLite por una base de datos en la nube mediante la interfaz de almacenamiento.

**Agradecimientos.** Se utilizaron herramientas de inteligencia artificial generativa como apoyo en el diseño, la implementación, la revisión del código y la redacción de este informe: **[AJUSTAR: indicar las herramientas y modelos realmente utilizados y para qué tareas]**. El autor revisó el contenido, comprende el código desarrollado y asume la responsabilidad total del trabajo presentado.

## Referencias

[1] National Institute of Standards and Technology, "Secure Hash Standard (SHS)," Federal Information Processing Standards Publication FIPS PUB 180-4, agosto de 2015.

[2] S. Nakamoto, "Bitcoin: A Peer-to-Peer Electronic Cash System," 2008.

[3] A. Narayanan, J. Bonneau, E. Felten, A. Miller y S. Goldfeder, *Bitcoin and Cryptocurrency Technologies*. Princeton, NJ, EE. UU.: Princeton University Press, 2016.

[4] L. Lamport, "Password authentication with insecure communication," *Communications of the ACM*, vol. 24, no. 11, pp. 770–772, 1981.

[5] S. Haber y W. S. Stornetta, "How to time-stamp a digital document," *Journal of Cryptology*, vol. 3, no. 2, pp. 99–111, 1991.