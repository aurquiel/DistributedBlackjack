## 21 Blackjack Distribuido

Juego cliente-servidor en Python (pygame + sockets) para 1-3 jugadores contra el crupier. El cliente incluye un botón “README” que abre esta documentación desde la interfaz.

### Reglas del juego

-   1. El objetivo es llegar a 21 sin pasarse.
-   2. J, Q, K valen 10; el As vale 1 u 11; el resto vale su número.
-   3. El crupier se planta entre 17 y 21.
-   4. Si empatas recuperas la apuesta; si ganas la duplicas.
-   5. Flujo de apuesta: recargar saldo → apostar → pedir carta o plantarse.
-   6. Solo puedes doblar la apuesta con exactamente dos cartas.
-   7. La partida inicia con 3 jugadores; si hay cupos, se pueden unir nuevos en medio de la partida (activos desde la siguiente ronda).

### Arquitectura y decisiones de diseño

- **Servidor autoritativo (fuente de verdad):** toda regla crítica vive en el servidor (saldo, apuestas, reparto, turnos, resultados). El cliente nunca decide quién gana; solo renderiza estado y envía acciones.
- **Modelo de hilos en servidor:**
	- 1 hilo acepta conexiones TCP nuevas.
	- 1 hilo por cada cliente recibe mensajes de ese socket (cada mensjae es un comando del juego) y lo coloca en una cola para ser consumido.
	- 1 hilo de juego consume todos los comandos (una lista usando lock para acceso exclusivo) y aplica la lógica de forma secuencial.
- **Modelo de hilos en cliente:**
	- 1 hilo de UI (pygame) para dibujar y capturar input.
	- 1 hilo de conexión creado al intentar conectar, para ejecutar `connect()` sin congelar la UI.
	- 1 hilo receptor de mensajes del servidor (recibe los comandos del servidor y los agrega a una lista usando lock para consumirlos en otro hilo, asi permanece escuchando sin ahcer atreas pesadas que puedan bloquear el hilo).
	- 1 hilo que procesa la cola de comandos y actualiza el estado local.
- **Colas thread-safe + locks:** cliente y servidor usan una cola protegida por `Lock` para desacoplar recepción de red y lógica del juego. En servidor, además se protege la lista de sockets/nombres conectados para evitar condiciones de carrera.
- **Protocolo TCP simple y robusto:** cada mensaje viaja como `header(10 bytes con longitud) + payload de texto`. Esto evita lecturas parciales y facilita parseo de comandos (`\n`, `\a`, `\h`, etc.).
- **Gestión de cupos y entradas tardías:** el máximo es 3 jugadores concurrentes. Si un cliente entra con partida llena recibe `\f`. Si entra con partida activa y hay cupo, queda sincronizado y participa plenamente desde la siguiente ronda.
- **Orquestación de rondas:** la ronda inicia cuando todos los jugadores conectados tienen apuesta > 0; el servidor reparte, rota turnos con `\x`, y al finalizar ejecuta automáticamente el turno del crupier hasta 17–21 o puede perder.
- **Consistencia de apuestas/saldo:** recarga (`\m`), apuesta (`\a`) y doblar (`\c`) se validan en servidor antes de difundir cambios, evitando desincronización entre clientes.
- **Tolerancia a fallos y desconexiones:** ante error de socket, servidor elimina y cierra al jugador, difunde `\u` y reasigna turno si corresponde. El cliente marca error de conexión, limpia estado local y permite reintentar.
- **Baraja y equilibrio de juego:** se usa una baraja de 4 mazos estándar (52 × 4), con reshuffle al limpiar ronda, para reducir patrones repetitivos en partidas largas.

### Protocolo de mensajes (exhaustivo)

- `\n <jugador>`: Solicitar unirse; el servidor responde con lista completa vía `\n ...` o con `\f` si está lleno.
- `\f`: Partida llena; el cliente muestra “Servidor Full”.
- `\y`: Señal de partida activa (inicio o aceptación tardía). El cliente marca la partida como lista.
- `\x <jugador>`: Otorga turno.
- `\z <jugador>`: Fin de turno; el servidor rota o finaliza ronda.
- `\m <jugador> <monto>`: Recarga de saldo.
- `\a <jugador> <monto> <balance>`: Apuesta aplicada; incluye balance actualizado.
- `\c <jugador> <nueva_apuesta> <balance>`: Doble apuesta aplicada.
- `\t`: Saldo insuficiente para apostar/doblar; el cliente muestra aviso.
- `\k <jugador> <c1> <c2>`: Reparto inicial de dos cartas al jugador.
- `\h <jugador> <carta...>`: Mano completa del jugador tras pedir carta.
- `\s <carta...>`: Cartas visibles del crupier.
- `\v <valor>`: Valor actual del crupier.
- `\w <jugador> <balance>`: El jugador gana la ronda; incluye nuevo balance.
- `\g <jugador> <balance>`: Empate.
- `\l <jugador> <balance>`: El jugador pierde.
- `\b`: Limpia mesa y apuestas para siguiente ronda.
- `\u <jugador>`: Jugador desconectado; el servidor lo quita y reasigna turno si es necesario.
- `\i <texto>`: Mensaje informativo interno del servidor (logs de conexiones, errores y eventos operativos).

### Flujo de comunicación (alto nivel)

1. Conexión: cliente envía `\n`; servidor acepta o responde `\f` si lleno.
2. Inicio de partida: cuando hay 3 jugadores, servidor emite `\y` y `\x` al primero.
3. Apuestas: clientes envían `\a`/`\c`; el servidor valida saldo y difunde el estado.
4. Acciones de turno: `\h` para pedir carta, `\z` para cerrar turno. El servidor rota con `\x`.
5. Cierre de ronda: tras todos los turnos, el crupier juega (`\s`, `\v`), se envían resultados (`\w`, `\g`, `\l`) y limpieza (`\b`).
6. Desconexiones: `\u` libera el cupo; si no hay ronda, el servidor asigna turno al siguiente disponible.

### Requisitos técnicos

- Python 3.9+ recomendado.
- Dependencias: `pygame` (cliente y servidor), `opencv-python` opcional para el video promocional.
- Red: los clientes deben alcanzar la IP/puerto del servidor (por defecto 12345 TCP).
- El juego tiene un resolución de 1200 X 800 px.

### Ejecución

1. (Opcional, recomendado) Crea y activa un entorno virtual:

```bash
python -m venv .venv
# Linux / macOS
source .venv/bin/activate

# Windows (PowerShell)
.venv\Scripts\Activate.ps1

# Windows (CMD)
.venv\Scripts\activate.bat
```

2. Instala dependencias:

```bash
pip install pygame opencv-python
```

3. Servidor:

```bash
cd Server
python main.py
#OR
python3 main.py
```

4. Cliente (uno por jugador):

```bash
cd Client
python main.py
#OR
python3 main.py
```

En el menú del cliente, configura IP/puerto si es necesario, usa “README” para abrir esta guía y pulsa “Iniciar Juego”.
