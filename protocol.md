GoBA Protocol for Unity Clients

Overview
- Transport: WebSocket (text JSON)
- HTTP helper endpoints: /create, /info
- Default port: 5000 (env PORT)
- Auto-created room on boot: ROOM (default TEST)

Endpoints
- Create Game (HTTP GET)
  - URL: http://HOST:PORT/create?name=YourName
  - Response JSON: { "code": string, "success": bool, "error": string }
- Join Game (WebSocket)
  - URL: ws://HOST:PORT/join?code=ROOM_CODE&name=YourName
  - Upon connect, server will emit a personal "connection" event followed by a personal "setup" event.
- Info (HTTP GET)
  - URL: http://HOST:PORT/info
  - Response JSON: { "liveGames": number, "livePlayers": number }

Message Envelopes
- Client → Server (ClientEvent)
  {
    "category": string,
    "event": string,
    "timestamp": number, // Unix seconds
    "data": any
  }
  Notes:
  - category must be "game" for gameplay inputs.
  - event names: "move", "shoot", "dash".

- Server → Client (ServerEvent)
  {
    "subscription": string,   // "personal" | "global-events" | "team-events"
    "name": string,           // event name, e.g., "connection", "setup", "update-teams", "tick"
    "timestamp": number,      // Unix seconds
    "data": any               // payload serialized as JSON
  }

Server Events and Payloads
- personal / connection
  {
    "success": boolean,
    "error": string
  }

- personal / setup
  SetupUpdate
  {
    "id": string,                 // your client UUID as string
    "walls": Rectangle[],
    "bushes": Rectangle[]
  }
  Rectangle
  {
    "x": number,
    "y": number,
    "w": number,
    "h": number
  }

- global-events / update-teams
  TeamsUpdate
  {
    "teams": { [teamName: string]: TeamJSON },
    "clients": { [clientId: string]: string /* teamName */ },
    "scores": { [playerName: string]: Score }
  }
  TeamJSON
  {
    "color": string,   // hex color e.g., "#ff0000"
    "size": number
  }
  Score
  {
    "kills": number,
    "deaths": number,
    "assists": number
  }

- team-events / tick
  TickUpdate
  {
    "champions": Champion[],
    "projectiles": Projectile[]
  }
  Champion
  {
    "id": string,      // UUID
    "name": string,
    "health": number,
    "r": number,       // radius
    "x": number,
    "y": number
  }
  Projectile
  {
    "team": string,    // team name
    "r": number,       // radius
    "x": number,
    "y": number
  }

Client Events (Unity → Server)
- category: "game"

- event: "move"
  data: { "x": number, "y": number }
  Notes: direction vector. Send on change or at fixed rate.

- event: "shoot"
  data: { "x": number, "y": number }
  Notes: aim/target world coords.

- event: "dash"
  data: {}

Unity Implementation Guidelines
- Transport:
  - Desktop/Mobile: WebSocketSharp or System.Net.WebSockets (where supported).
  - WebGL: use browser WebSocket via a WebGL-compatible plugin.
- Deserialize:
  - First parse ServerEvent envelope; switch on (subscription, name), then parse data into the appropriate payload type.
- State Management:
  - Store your own clientId from SetupUpdate.id.
  - Maintain maps:
    - clientId → teamName (from update-teams.clients)
    - teamName → TeamJSON (color, size)
    - playerName → Score
  - For tick: update dictionaries for champions (by id) and projectiles (rebuild each tick).
- Rendering:
  - Champions: render with size r and color by team.
  - Projectiles: small spheres with r. Only visible entities included per team vision.
- Timing/Interpolation (optional):
  - Use ServerEvent.timestamp (Unix seconds) to buffer and interpolate.

Examples
- Client move event
  {
    "category": "game",
    "event": "move",
    "timestamp": 1710000000,
    "data": { "x": 1, "y": 0 }
  }

- Sample server tick envelope
  {
    "subscription": "team-events",
    "name": "tick",
    "timestamp": 1710000001,
    "data": { "champions": [ { "id": "...", "name": "Bob", "health": 100, "r": 30, "x": 1500, "y": 400 } ], "projectiles": [] }
  }

Notes
- CORS/Origin are permissive by default in the server code.
- For production (mobile/desktop), terminate TLS for wss://.
- Default game auto-spawn: code ROOM, default TEST. You can join directly without calling /create during development.
