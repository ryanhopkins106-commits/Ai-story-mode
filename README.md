# Ai-story-mode
Ai interactive story creations 
const express = require('express');
const cors = require('cors');
const dotenv = require('dotenv');
const mongoose = require('mongoose');
const storyRoutes = require('./routes/stories');
const narrativeRoutes = require('./routes/narrative');

dotenv.config();

const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Database Connection
mongoose.connect(process.env.MONGODB_URI || 'mongodb://localhost:27017/story-app')
  .then(() => console.log('MongoDB connected'))
  .catch(err => console.log('MongoDB connection error:', err));

// Routes
app.use('/api/stories', storyRoutes);
app.use('/api/narrative', narrativeRoutes);

// Error handling
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Something went wrong!' });
});

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));

module.exports = app;
const mongoose = require('mongoose');

const storySchema = new mongoose.Schema({
  title: {
    type: String,
    required: true,
  },
  author: {
    type: String,
    required: true,
  },
  description: String,
  genre: String,
  initialContent: String,
  narrativeGraph: {
    type: Map,
    of: {
      content: String,
      choices: [{
        userInput: String, // Pattern or exact match
        nextNodeId: String,
        condition: String, // Optional: custom logic
      }],
    },
  },
  createdAt: {
    type: Date,
    default: Date.now,
  },
  updatedAt: Date,
  viewCount: {
    type: Number,
    default: 0,
  },
  rating: {
    type: Number,
    default: 0,
  },
});

module.exports = mongoose.model('Story', storySchema);const mongoose = require('mongoose');

const sessionSchema = new mongoose.Schema({
  storyId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Story',
    required: true,
  },
  userId: String, // Anonymous or authenticated user
  currentNodeId: String,
  history: [{
    nodeId: String,
    content: String,
    userInput: String,
    timestamp: Date,
  }],
  choices: [{
    input: String,
    outcome: String,
  }],
  createdAt: {
    type: Date,
    default: Date.now,
  },
  updatedAt: Date,
});

module.exports = mongoose.model('NarrativeSession', sessionSchema);const express = require('express');
const router = express.Router();
const Story = require('../models/Story');

// Get all stories
router.get('/', async (req, res) => {
  try {
    const stories = await Story.find().select('-narrativeGraph');
    res.json(stories);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get single story
router.get('/:id', async (req, res) => {
  try {
    const story = await Story.findById(req.params.id);
    if (!story) return res.status(404).json({ error: 'Story not found' });
    story.viewCount += 1;
    await story.save();
    res.json(story);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create new story
router.post('/', async (req, res) => {
  try {
    const story = new Story(req.body);
    await story.save();
    res.status(201).json(story);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Update story
router.put('/:id', async (req, res) => {
  try {
    const story = await Story.findByIdAndUpdate(req.params.id, req.body, { new: true });
    res.json(story);
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

module.exports = router;const express = require('express');
const router = express.Router();
const NarrativeSession = require('../models/NarrativeSession');
const Story = require('../models/Story');
const { processNarrativeInput } = require('../utils/narrativeEngine');

// Start a new narrative session
router.post('/start/:storyId', async (req, res) => {
  try {
    const story = await Story.findById(req.params.storyId);
    if (!story) return res.status(404).json({ error: 'Story not found' });

    const session = new NarrativeSession({
      storyId: req.params.storyId,
      userId: req.body.userId || `anon_${Date.now()}`,
      currentNodeId: 'start',
      history: [{
        nodeId: 'start',
        content: story.initialContent,
        timestamp: new Date(),
      }],
    });

    await session.save();
    res.status(201).json({
      sessionId: session._id,
      currentNodeId: 'start',
      content: story.initialContent,
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Process user input and advance narrative
router.post('/process/:sessionId', async (req, res) => {
  try {
    const { userInput } = req.body;
    const session = await NarrativeSession.findById(req.params.sessionId);
    const story = await Story.findById(session.storyId);

    if (!session || !story) {
      return res.status(404).json({ error: 'Session or story not found' });
    }

    // Process the narrative input
    const result = await processNarrativeInput(
      userInput,
      story,
      session.currentNodeId
    );

    // Update session
    session.history.push({
      nodeId: session.currentNodeId,
      content: result.nextContent,
      userInput: userInput,
      timestamp: new Date(),
    });

    session.choices.push({
      input: userInput,
      outcome: result.outcome,
    });

    session.currentNodeId = result.nextNodeId;
    session.updatedAt = new Date();
    await session.save();

    res.json({
      nextNodeId: result.nextNodeId,
      content: result.nextContent,
      outcome: result.outcome,
      sessionId: session._id,
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get session history
router.get('/:sessionId', async (req, res) => {
  try {
    const session = await NarrativeSession.findById(req.params.sessionId);
    res.json(session);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;const natural = require('natural');

/**
 * Processes user input and determines next narrative outcome
 * Supports pattern matching and AI-based semantic analysis
 */
async function processNarrativeInput(userInput, story, currentNodeId) {
  const currentNode = story.narrativeGraph.get(currentNodeId);
  
  if (!currentNode) {
    return {
      nextNodeId: currentNodeId,
      nextContent: "Story path not found.",
      outcome: "error",
    };
  }

  const input = userInput.toLowerCase().trim();
  let matchedChoice = null;

  // 1. Try exact matching
  for (const choice of currentNode.choices) {
    if (input === choice.userInput.toLowerCase()) {
      matchedChoice = choice;
      break;
    }
  }

  // 2. Try keyword matching if no exact match
  if (!matchedChoice) {
    for (const choice of currentNode.choices) {
      if (matchesPattern(input, choice.userInput)) {
        matchedChoice = choice;
        break;
      }
    }
  }

  // 3. If still no match, evaluate semantic similarity
  if (!matchedChoice) {
    matchedChoice = findMostSimilarChoice(input, currentNode.choices);
  }

  const nextNodeId = matchedChoice?.nextNodeId || 'default_end';
  const nextNode = story.narrativeGraph.get(nextNodeId) || {
    content: "Your story has ended. Thank you for playing!",
  };

  return {
    nextNodeId,
    nextContent: nextNode.content,
    outcome: matchedChoice?.userInput || "Your choice had an unexpected outcome.",
  };
}

/**
 * Simple pattern matching for keywords
 */
function matchesPattern(input, pattern) {
  const keywords = pattern.toLowerCase().split(/\s+/);
  return keywords.every(keyword => input.includes(keyword));
}

/**
 * Find most similar choice using Levenshtein distance
 */
function findMostSimilarChoice(input, choices) {
  let bestMatch = null;
  let highestSimilarity = 0;

  for (const choice of choices) {
    const similarity = calculateSimilarity(input, choice.userInput.toLowerCase());
    if (similarity > highestSimilarity) {
      highestSimilarity = similarity;
      bestMatch = choice;
    }
  }

  return bestMatch || choices[0];
}

/**
 * Calculate similarity between two strings (0-1)
 */
function calculateSimilarity(str1, str2) {
  const longer = str1.length > str2.length ? str1 : str2;
  const shorter = str1.length > str2.length ? str2 : str1;

  if (longer.length === 0) return 1.0;

  const editDistance = getEditDistance(longer, shorter);
  return (longer.length - editDistance) / longer.length;
}

/**
 * Levenshtein distance algorithm
 */
function getEditDistance(s1, s2) {
  const costs = [];
  for (let k = 0; k <= s1.length; k++) {
    let lastValue = k;
    for (let i = 0; i <= s2.length; i++) {
      if (k === 0) {
        costs[i] = i;
      } else if (i > 0) {
        let newValue = costs[i - 1];
        if (s1.charAt(k - 1) !== s2.charAt(i - 1)) {
          newValue = Math.min(Math.min(newValue, lastValue), costs[i]) + 1;
        }
        costs[i - 1] = lastValue;
        lastValue = newValue;
      }
    }
    if (k > 0) costs[s2.length] = lastValue;
  }
  return costs[s2.length];
}

module.exports = { processNarrativeInput };{
  "name": "interactive-story-app",
  "version": "1.0.0",
  "description": "An interactive storytelling platform with free-form text input",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "mongoose": "^7.0.0",
    "dotenv": "^16.0.3",
    "natural": "^6.7.0"
  },
  "devDependencies": {
    "nodemon": "^2.0.20",
    "jest": "^29.5.0"
  }
}PORT=5000
MONGODB_URI=mongodb://localhost:27017/story-app
NODE_ENV=developmentimport React, { useState, useEffect } from 'react';
import axios from 'axios';
import './StoryPlayer.css';

function StoryPlayer({ storyId }) {
  const [sessionId, setSessionId] = useState(null);
  const [currentContent, setCurrentContent] = useState('');
  const [userInput, setUserInput] = useState('');
  const [history, setHistory] = useState([]);
  const [loading, setLoading] = useState(false);

  const API_BASE = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';

  // Start story session
  useEffect(() => {
    const startStory = async () => {
      try {
        setLoading(true);
        const response = await axios.post(
          `${API_BASE}/narrative/start/${storyId}`,
          { userId: `user_${Date.now()}` }
        );
        setSessionId(response.data.sessionId);
        setCurrentContent(response.data.content);
        setHistory([response.data.content]);
      } catch (error) {
        console.error('Error starting story:', error);
      } finally {
        setLoading(false);
      }
    };

    startStory();
  }, [storyId]);

  // Handle user input submission
  const handleSubmit = async (e) => {
    e.preventDefault();

    if (!userInput.trim() || !sessionId) return;

    try {
      setLoading(true);
      const response = await axios.post(
        `${API_BASE}/narrative/process/${sessionId}`,
        { userInput }
      );

      setCurrentContent(response.data.content);
      setHistory([...history, `You: ${userInput}`, response.data.content]);
      setUserInput('');
    } catch (error) {
      console.error('Error processing narrative:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="story-player">
      <div className="story-content">
        <p>{currentContent}</p>
      </div>

      <div className="story-history">
        {history.map((entry, idx) => (
          <p key={idx} className="history-entry">
            {entry}
          </p>
        ))}
      </div>

      <form onSubmit={handleSubmit} className="input-form">
        <input
          type="text"
          value={userInput}
          onChange={(e) => setUserInput(e.target.value)}
          placeholder="What do you do? (free-form text)"
          disabled={loading}
        />
        <button type="submit" disabled={loading}>
          {loading ? 'Processing...' : 'Continue'}
        </button>
      </form>
    </div>
  );
}

export default StoryPlayer;.story-player {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Georgia', serif;
}

.story-content {
  background: #f5f5f5;
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 20px;
  min-height: 200px;
  line-height: 1.6;
}

.story-history {
  max-height: 300px;
  overflow-y: auto;
  background: #fff;
  border: 1px solid #ddd;
  padding: 15px;
  margin-bottom: 20px;
  border-radius: 8px;
}

.history-entry {
  margin: 8px 0;
  padding: 5px 0;
  border-bottom: 1px solid #eee;
}

.input-form {
  display: flex;
  gap: 10px;
}

.input-form input {
  flex: 1;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 16px;
}

.input-form button {
  padding: 12px 24px;
  background: #007bff;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
}

.input-form button:hover {
  background: #0056b3;
}

.input-form button:disabled {
  background: #ccc;
  cursor: not-allowed;
}# Interactive Story App

An interactive storytelling platform where users can influence narrative outcomes with free-form text input.

## Features

- 📖 **Interactive Narratives**: Create branching stories with multiple outcomes
- 🎯 **Free-Form Input**: Users can type any action/choice to influence the story
- 🤖 **Smart Matching**: Pattern matching and semantic similarity for natural interactions
- 📊 **Session History**: Track user choices and story progression
- 🔄 **Replayability**: Multiple playthroughs create different experiences
- 📈 **Leaderboard**: Monthly top stories by views and ratings

## Tech Stack

- **Backend**: Node.js + Express
- **Database**: MongoDB
- **Frontend**: React
- **NLP**: Natural.js for text processing

## Setup Instructions

### Prerequisites
- Node.js 14+
- MongoDB
- npm or yarn

### Backend Setup

```bash
# Install dependencies
npm install

# Create .env file
cp .env.example .env

# Start server
npm run devcd client
npm install
npm start
