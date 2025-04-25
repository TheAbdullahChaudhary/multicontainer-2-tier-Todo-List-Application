# Multi-Container 2-Tier Todo List Application

A full-stack todo list application built with:
- Frontend: React.js with Nginx
- Backend: Node.js with Express
- Database: MongoDB
- Containerization: Docker

## Features
- Create, read, update, and delete todo items
- Mark todos as complete/incomplete
- Containerized architecture for easy deployment
- Multi-container setup with dedicated services

## Architecture
- **Frontend Container**: React.js with Nginx as a reverse proxy
- **Backend Container**: Node.js/Express REST API
- **Database Container**: MongoDB for data storage

## Prerequisites
- Docker installed on your machine
- Git for cloning the repository

## Step-by-Step Setup Guide

### 1. Clone the Repository
```bash
git clone https://github.com/TheAbdullahChaudhary/multicontainer-2-tier-Todo-List-Application.git
cd multicontainer-2-tier-Todo-List-Application
```

### 2. Create Docker Network
```bash
docker network create todo-app-network
```

### 3. Set Up MongoDB Container
```bash
docker run -d \
  --name todo-mongo \
  --network todo-app-network \
  -p 27017:27017 \
  -v mongo-data:/data/db \
  mongo:latest
```

### 4. Set Up Backend Container

#### Backend Code (index.js)
```javascript
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const dotenv = require('dotenv');

// Load environment variables
dotenv.config();

const app = express();
const PORT = process.env.PORT || 3000;
const MONGODB_URI = process.env.MONGODB_URI || 'mongodb://todo-mongo:27017/todos';

// Middleware
app.use(cors({
  origin: '*',  // Allow all origins for now
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  credentials: true
}));
app.use(express.json());

// Connect to MongoDB
mongoose.connect(MONGODB_URI)
  .then(() => console.log('Connected to MongoDB'))
  .catch(err => {
    console.error('MongoDB connection error:', err);
    process.exit(1);
  });

// Todo Schema and Model
const todoSchema = new mongoose.Schema({
  title: { type: String, required: true },
  completed: { type: Boolean, default: false },
  createdAt: { type: Date, default: Date.now }
});

const Todo = mongoose.model('Todo', todoSchema);

// Routes
// GET all todos
app.get('/todos', async (req, res) => {
  try {
    const todos = await Todo.find().sort({ createdAt: -1 });
    res.json(todos);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// GET a single todo
app.get('/todos/:id', async (req, res) => {
  try {
    const todo = await Todo.findById(req.params.id);
    if (!todo) {
      return res.status(404).json({ message: 'Todo not found' });
    }
    res.json(todo);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// CREATE a todo
app.post('/todos', async (req, res) => {
  try {
    const { title } = req.body;
    if (!title) {
      return res.status(400).json({ message: 'Title is required' });
    }
    
    const newTodo = new Todo({ title });
    const savedTodo = await newTodo.save();
    res.status(201).json(savedTodo);
  } catch (err) {
    res.status(400).json({ message: err.message });
  }
});

// UPDATE a todo
app.put('/todos/:id', async (req, res) => {
  try {
    const { title, completed } = req.body;
    const updatedTodo = await Todo.findByIdAndUpdate(
      req.params.id,
      { title, completed },
      { new: true }
    );
    
    if (!updatedTodo) {
      return res.status(404).json({ message: 'Todo not found' });
    }
    
    res.json(updatedTodo);
  } catch (err) {
    res.status(400).json({ message: err.message });
  }
});

// DELETE a todo
app.delete('/todos/:id', async (req, res) => {
  try {
    const deletedTodo = await Todo.findByIdAndDelete(req.params.id);
    
    if (!deletedTodo) {
      return res.status(404).json({ message: 'Todo not found' });
    }
    
    res.json({ message: 'Todo deleted' });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// Health check endpoint
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok' });
});

// Root endpoint for testing
app.get('/', (req, res) => {
  res.json({ message: 'Todo API is running' });
});

// Start server
app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server running on port ${PORT}`);
});
```

#### Backend Setup Commands
```bash
cd backend

# Create package.json
cat > package.json << 'EOF'
{
  "name": "todo-backend",
  "version": "1.0.0",
  "description": "Todo List Backend API",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.0.3",
    "express": "^4.18.2",
    "mongoose": "^7.1.0"
  }
}
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["node", "index.js"]
EOF

# Build and run backend container
docker build -t todo-backend .

docker run -d \
  --name todo-backend \
  --network todo-app-network \
  -p 3000:3000 \
  -e MONGODB_URI=mongodb://todo-mongo:27017/todos \
  -e PORT=3000 \
  todo-backend
```

### 5. Set Up Frontend Container

#### Frontend Nginx Configuration
```bash
cd ../frontend

# Create nginx.conf
cat > nginx.conf << 'EOF'
server {
    listen 80;
    
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }
    
    location /api/ {
        rewrite ^/api/(.*) /$1 break;
        proxy_pass http://todo-backend:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
EOF
```

#### Frontend App.js Code
```jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './App.css';

// Use API path which will be routed through Nginx
const API_URL = '/api';

function App() {
  const [todos, setTodos] = useState([]);
  const [newTodo, setNewTodo] = useState('');
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  // Fetch todos
  const fetchTodos = async () => {
    try {
      setLoading(true);
      const response = await axios.get(`${API_URL}/todos`);
      setTodos(response.data);
      setError(null);
    } catch (err) {
      console.error('Error fetching todos:', err);
      setError('Failed to fetch todos');
    } finally {
      setLoading(false);
    }
  };

  // Add todo
  const addTodo = async (e) => {
    e.preventDefault();
    if (!newTodo.trim()) return;

    try {
      const response = await axios.post(`${API_URL}/todos`, { title: newTodo });
      setTodos([response.data, ...todos]);
      setNewTodo('');
      setError(null);
    } catch (err) {
      console.error('Error adding todo:', err);
      setError('Failed to add todo');
    }
  };

  // Toggle todo completion
  const toggleTodo = async (id, completed) => {
    try {
      const todo = todos.find(t => t._id === id);
      const response = await axios.put(`${API_URL}/todos/${id}`, {
        title: todo.title,
        completed: !completed
      });
      
      setTodos(todos.map(todo => 
        todo._id === id ? response.data : todo
      ));
      setError(null);
    } catch (err) {
      console.error('Error updating todo:', err);
      setError('Failed to update todo');
    }
  };

  // Delete todo
  const deleteTodo = async (id) => {
    try {
      await axios.delete(`${API_URL}/todos/${id}`);
      setTodos(todos.filter(todo => todo._id !== id));
      setError(null);
    } catch (err) {
      console.error('Error deleting todo:', err);
      setError('Failed to delete todo');
    }
  };

  // Load todos on component mount
  useEffect(() => {
    fetchTodos();
  }, []);

  return (
    <div className="app">
      <header>
        <h1>Todo List</h1>
      </header>
      
      <form onSubmit={addTodo} className="add-form">
        <input
          type="text"
          value={newTodo}
          onChange={(e) => setNewTodo(e.target.value)}
          placeholder="Add a new task..."
        />
        <button type="submit">Add</button>
      </form>

      {error && <div className="error">{error}</div>}
      
      {loading ? (
        <p>Loading todos...</p>
      ) : (
        <ul className="todo-list">
          {todos.length === 0 ? (
            <p>No todos yet. Add one above!</p>
          ) : (
            todos.map(todo => (
              <li key={todo._id} className={todo.completed ? 'completed' : ''}>
                <span 
                  onClick={() => toggleTodo(todo._id, todo.completed)}
                  className="todo-title"
                >
                  {todo.title}
                </span>
                <div className="actions">
                  <button 
                    onClick={() => toggleTodo(todo._id, todo.completed)}
                    className="toggle-btn"
                  >
                    {todo.completed ? 'Undo' : 'Complete'}
                  </button>
                  <button 
                    onClick={() => deleteTodo(todo._id)}
                    className="delete-btn"
                  >
                    Delete
                  </button>
                </div>
              </li>
            ))
          )}
        </ul>
      )}
    </div>
  );
}

export default App;
```

#### Frontend Dockerfile
```bash
# Create Dockerfile for frontend
cat > Dockerfile << 'EOF'
FROM node:18-alpine as build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

# Build and run frontend container
docker build -t todo-frontend .

docker run -d \
  --name todo-frontend \
  --network todo-app-network \
  -p 80:80 \
  todo-frontend
```

### 6. Creating a Docker Compose File (Optional)

For easier management, you can create a docker-compose.yml file:

```bash
cd ..
cat > docker-compose.yml << 'EOF'
version: '3'

services:
  mongodb:
    image: mongo:latest
    container_name: todo-mongo
    volumes:
      - mongo-data:/data/db
    ports:
      - "27017:27017"
    networks:
      - todo-app-network

  backend:
    build: ./backend
    container_name: todo-backend
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/todos
      - PORT=3000
    ports:
      - "3000:3000"
    depends_on:
      - mongodb
    networks:
      - todo-app-network

  frontend:
    build: ./frontend
    container_name: todo-frontend
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - todo-app-network

networks:
  todo-app-network:
    driver: bridge

volumes:
  mongo-data:
EOF

# Start all services with Docker Compose
docker-compose up -d
```

## Troubleshooting

### Common Issues and Solutions

1. **Frontend can't connect to backend**
   - Check Nginx configuration in the frontend container
   - Verify network connectivity: `docker exec todo-frontend ping todo-backend`
   - Check backend logs: `docker logs todo-backend`

2. **MongoDB connection issues**
   - Check MongoDB container status: `docker ps`
   - Verify MongoDB logs: `docker logs todo-mongo`
   - Make sure the URL in the backend code is correct: `mongodb://todo-mongo:27017/todos`

3. **Container startup failures**
   - Check container logs: `docker logs <container-name>`
   - Verify required ports are available on your host machine

4. **API errors**
   - Test backend API directly: `curl http://localhost:3000/todos`
   - Check browser console for CORS errors

### Checking Container Logs
```bash
# Check frontend logs
docker logs todo-frontend

# Check backend logs
docker logs todo-backend

# Check MongoDB logs
docker logs todo-mongo
```

### Testing APIs Directly
```bash
# Get all todos
curl http://localhost:3000/todos

# Create a new todo
curl -X POST -H "Content-Type: application/json" -d '{"title":"Test Todo"}' http://localhost:3000/todos

# Update a todo (replace <todo-id> with actual ID)
curl -X PUT -H "Content-Type: application/json" -d '{"title":"Updated Todo","completed":true}' http://localhost:3000/todos/<todo-id>

# Delete a todo (replace <todo-id> with actual ID)
curl -X DELETE http://localhost:3000/todos/<todo-id>
```

## Deployment Options

### On EC2/Cloud Instance
1. Make sure Docker is installed on your instance
2. Clone the repository
3. Follow the setup instructions above
4. Ensure ports 80 and 3000 are open in your security group/firewall

### Using Docker Swarm
```bash
# Initialize swarm
docker swarm init

# Deploy the stack
docker stack deploy -c docker-compose.yml todo-app
```

## Cleanup
To stop and remove all containers, networks, and volumes:

```bash
# Stop individual containers
docker stop todo-frontend todo-backend todo-mongo

# Remove containers
docker rm todo-frontend todo-backend todo-mongo

# Remove network
docker network rm todo-app-network

# Remove volumes
docker volume rm mongo-data

# Or using Docker Compose
docker-compose down -v
```

## Contributing
1. Fork the repository
2. Create your feature branch: `git checkout -b feature/my-new-feature`
3. Commit your changes: `git commit -am 'Add some feature'`
4. Push to the branch: `git push origin feature/my-new-feature`
5. Submit a pull request


```

Project URL: https://github.com/TheAbdullahChaudhary/multicontainer-2-tier-Todo-List-Application.git
