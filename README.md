/* =========================
   Backend (server.js)
   ========================= */

const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
require('dotenv').config();

const app = express();
app.use(express.json());
app.use(cors());

mongoose.connect(process.env.MONGO_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});

const UserSchema = new mongoose.Schema({
  username: String,
  password: String,
});
const TaskSchema = new mongoose.Schema({
  userId: String,
  title: String,
  completed: Boolean,
});

const User = mongoose.model('User', UserSchema);
const Task = mongoose.model('Task', TaskSchema);

app.post('/register', async (req, res) => {
  const { username, password } = req.body;
  const hashedPassword = await bcrypt.hash(password, 10);
  const user = new User({ username, password: hashedPassword });
  await user.save();
  res.json({ message: 'User registered' });
});

app.post('/login', async (req, res) => {
  const { username, password } = req.body;
  const user = await User.findOne({ username });
  if (user && (await bcrypt.compare(password, user.password))) {
    const token = jwt.sign({ userId: user._id }, process.env.JWT_SECRET);
    res.json({ token });
  } else {
    res.status(401).json({ message: 'Invalid credentials' });
  }
});

const authMiddleware = (req, res, next) => {
  const token = req.headers.authorization;
  if (!token) return res.status(403).json({ message: 'No token provided' });
  jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
    if (err) return res.status(403).json({ message: 'Invalid token' });
    req.userId = decoded.userId;
    next();
  });
};

app.post('/tasks', authMiddleware, async (req, res) => {
  const task = new Task({ userId: req.userId, title: req.body.title, completed: false });
  await task.save();
  res.json(task);
});

app.get('/tasks', authMiddleware, async (req, res) => {
  const tasks = await Task.find({ userId: req.userId });
  res.json(tasks);
});

app.put('/tasks/:id', authMiddleware, async (req, res) => {
  await Task.findByIdAndUpdate(req.params.id, req.body);
  res.json({ message: 'Task updated' });
});

app.delete('/tasks/:id', authMiddleware, async (req, res) => {
  await Task.findByIdAndDelete(req.params.id);
  res.json({ message: 'Task deleted' });
});

app.listen(5000, () => console.log('Server running on port 5000'));

/* =========================
   Frontend (App.js)
   ========================= */

import { useEffect, useState } from 'react';

export default function App() {
  const [tasks, setTasks] = useState([]);
  const [task, setTask] = useState('');
  const token = localStorage.getItem('token');

  useEffect(() => {
    fetch('http://localhost:5000/tasks', { headers: { Authorization: token } })
      .then(res => res.json())
      .then(setTasks);
  }, [tasks]);

  const addTask = async () => {
    await fetch('http://localhost:5000/tasks', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: token },
      body: JSON.stringify({ title: task }),
    });
    setTask('');
  };

  return (
    <div className='p-5'>
      <h1 className='text-xl font-bold'>Task Manager</h1>
      <input
        className='border p-2 mr-2'
        value={task}
        onChange={(e) => setTask(e.target.value)}
      />
      <button className='bg-blue-500 text-white p-2' onClick={addTask}>Add Task</button>
      <ul>
        {tasks.map((t) => (
          <li key={t._id} className='mt-2'>{t.title}</li>
        ))}
      </ul>
    </div>
  );
}
