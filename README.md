/** @jsxImportSource https://esm.sh/react@18.2.0 */
import React, { useState, useEffect, useRef } from "https://esm.sh/react@18.2.0";
import { createRoot } from "https://esm.sh/react-dom@18.2.0/client";

function App() {
  const [messages, setMessages] = useState([]);
  const [inputMessage, setInputMessage] = useState("");
  const [username, setUsername] = useState("");
  const [isLoggedIn, setIsLoggedIn] = useState(false);
  const socketRef = useRef(null);

  useEffect(() => {
    if (isLoggedIn) {
      socketRef.current = new WebSocket(`wss://${location.host}/ws`);
      
      socketRef.current.onmessage = (event) => {
        const newMessage = JSON.parse(event.data);
        setMessages(prev => [...prev, newMessage]);
      };

      return () => {
        if (socketRef.current) socketRef.current.close();
      };
    }
  }, [isLoggedIn]);

  const sendMessage = () => {
    if (inputMessage.trim() && socketRef.current) {
      const messageObj = {
        text: inputMessage,
        sender: username,
        timestamp: new Date().toISOString()
      };
      socketRef.current.send(JSON.stringify(messageObj));
      setInputMessage("");
    }
  };

  const handleLogin = () => {
    if (username.trim()) {
      setIsLoggedIn(true);
    }
  };

  if (!isLoggedIn) {
    return (
      <div style={styles.loginContainer}>
        <h2>Enter Username</h2>
        <input
          type="text"
          value={username}
          onChange={(e) => setUsername(e.target.value)}
          style={styles.input}
        />
        <button onClick={handleLogin} style={styles.button}>
          Join Chat
        </button>
      </div>
    );
  }

  return (
    <div style={styles.container}>
      <div style={styles.header}>
        <h1>🔬 WhatsApp Clone</h1>
        <span>{username}</span>
      </div>
      <div style={styles.messageContainer}>
        {messages.map((msg, index) => (
          <div 
            key={index} 
            style={{
              ...styles.message,
              alignSelf: msg.sender === username ? 'flex-end' : 'flex-start',
              backgroundColor: msg.sender === username ? '#dcf8c6' : '#fff'
            }}
          >
            <strong>{msg.sender}</strong>
            <p>{msg.text}</p>
            <small>{new Date(msg.timestamp).toLocaleTimeString()}</small>
          </div>
        ))}
      </div>
      <div style={styles.inputContainer}>
        <input
          type="text"
          value={inputMessage}
          onChange={(e) => setInputMessage(e.target.value)}
          style={styles.input}
          onKeyPress={(e) => e.key === 'Enter' && sendMessage()}
        />
        <button onClick={sendMessage} style={styles.button}>
          Send
        </button>
      </div>
      <a 
        href={import.meta.url.replace("esm.town", "val.town")} 
        target="_top" 
        style={styles.sourceLink}
      >
        View Source
      </a>
    </div>
  );
}

const styles = {
  loginContainer: {
    display: 'flex',
    flexDirection: 'column',
    alignItems: 'center',
    justifyContent: 'center',
    height: '100vh',
    fontFamily: 'Arial, sans-serif'
  },
  container: {
    display: 'flex',
    flexDirection: 'column',
    height: '100vh',
    fontFamily: 'Arial, sans-serif',
    maxWidth: '600px',
    margin: '0 auto'
  },
  header: {
    display: 'flex',
    justifyContent: 'space-between',
    padding: '10px',
    backgroundColor: '#f0f0f0'
  },
  messageContainer: {
    flex: 1,
    overflowY: 'auto',
    display: 'flex',
    flexDirection: 'column',
    padding: '10px'
  },
  message: {
    maxWidth: '70%',
    margin: '5px',
    padding: '10px',
    borderRadius: '10px',
    display: 'flex',
    flexDirection: 'column'
  },
  inputContainer: {
    display: 'flex',
    padding: '10px'
  },
  input: {
    flex: 1,
    padding: '10px',
    marginRight: '10px'
  },
  button: {
    padding: '10px',
    backgroundColor: '#25D366',
    color: 'white',
    border: 'none',
    borderRadius: '5px'
  },
  sourceLink: {
    position: 'absolute',
    bottom: '10px',
    right: '10px',
    color: '#888',
    textDecoration: 'none'
  }
};

function client() {
  createRoot(document.getElementById("root")).render(<App />);
}
if (typeof document !== "undefined") { client(); }

export default async function server(request: Request): Promise<Response> {
  const { sqlite } = await import("https://esm.town/v/stevekrouse/sqlite");
  const KEY = new URL(import.meta.url).pathname.split("/").at(-1);

  // WebSocket handling
  if (request.headers.get("upgrade") === "websocket") {
    const { socket, response } = Deno.upgradeWebSocket(request);
    const clients = new Set();

    socket.onopen = () => {
      clients.add(socket);
    };

    socket.onmessage = async (event) => {
      const message = JSON.parse(event.data);
      
      // Store message in SQLite
      await sqlite.execute(`
        CREATE TABLE IF NOT EXISTS ${KEY}_messages (
          id INTEGER PRIMARY KEY AUTOINCREMENT,
          sender TEXT,
          text TEXT,
          timestamp DATETIME
        )
      `);

      await sqlite.execute(`
        INSERT INTO ${KEY}_messages (sender, text, timestamp) 
        VALUES (?, ?, ?)
      `, [message.sender, message.text, message.timestamp]);

      // Broadcast to all connected clients
      for (const client of clients) {
        if (client.readyState === WebSocket.OPEN) {
          client.send(event.data);
        }
      }
    };

    socket.onclose = () => {
      clients.delete(socket);
    };

    return response;
  }

  return new Response(`
    <html>
      <head>
        <title>WhatsApp Clone</title>
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <script src="https://esm.town/v/std/catch"></script>
      </head>
      <body>
        <div id="root"></div>
        <script type="module" src="${import.meta.url}"></script>
      </body>
    </html>
  `, {
    headers: { "Content-Type": "text/html" }
  });
}
