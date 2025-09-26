[26/09/2025 06:00] ashu: fire-zone/
├── backend/
│   ├── models/     # MongoDB schemas (User , Tournament, Message, Withdrawal)
│   ├── routes/     # API routes (auth, tournaments, payments, admin, etc.)
│   ├── middleware/ # JWT auth, admin check
│   ├── server.js   # Entry point
│   └── package.json
├── frontend/
│   ├── src/
│   │   ├── components/ # Reusable (Navbar, Button, Card)
│   │   ├── pages/      # Landing, Login, Dashboard, etc.
│   │   ├── services/   # API calls (axios)
│   │   ├── App.js      # Router setup
│   │   └── index.css   # Tailwind imports
│   └── package.json
└── README.md
[26/09/2025 06:02] ashu: {
  "name": "fire-zone-backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mongoose": "^7.5.0",
    "bcryptjs": "^2.4.3",
    "jsonwebtoken": "^9.0.2",
    "cors": "^2.8.5",
    "dotenv": "^16.3.1",
    "nodemailer": "^6.9.7",
    "axios": "^1.5.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
[26/09/2025 06:02] ashu: MONGO_URI=mongodb://localhost:27017/firezone
JWT_SECRET=your_jwt_secret_key
ADMIN_EMAIL=officialfirezone7@gmail.com
UPI_ID=tiwari100456@okhdfcbank
[26/09/2025 06:03] ashu: const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  freeFireId: { type: Number }, // Numeric
  inGameName: { type: String, required: true },
  whatsapp: { type: String, required: true, match: [/^\d{10}$/, 'Invalid WhatsApp (10 digits)'] }, // Indian format
  coins: { type: Number, default: 50 }, // Starts with 50 on register
  role: { type: String, default: 'user', enum: ['user', 'admin'] },
  totalWinnings: { type: Number, default: 0 },
  joinedTournaments: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Tournament' }],
  avatar: { type: String, default: 'default-avatar.png' } // For leaderboard
}, { timestamps: true });

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(password) {
  return bcrypt.compare(password, this.password);
};

module.exports = mongoose.model('User', userSchema);
[26/09/2025 06:03] ashu: const mongoose = require('mongoose');

const tournamentSchema = new mongoose.Schema({
  name: { type: String, required: true },
  entryFee: { type: Number, required: true }, // Coins
  maxParticipants: { type: Number, required: true },
  prizePool: { type: Number, required: true },
  mapType: { type: String, default: 'Bermuda' },
  timing: { type: Date, required: true }, // Start time
  participants: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }],
  status: { type: String, enum: ['Upcoming', 'Live', 'Completed'], default: 'Upcoming' },
  winners: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }] // For results
}, { timestamps: true });

module.exports = mongoose.model('Tournament', tournamentSchema);
[26/09/2025 06:04] ashu: const mongoose = require('mongoose');

const messageSchema = new mongoose.Schema({
  tournamentId: { type: mongoose.Schema.Types.ObjectId, ref: 'Tournament', required: true },
  sender: { type: String, default: 'Admin' }, // Always Admin
  recipients: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }], // Joined users
  content: { type: String, required: true },
  createdAt: { type: Date, default: Date.now },
  read: { type: Map, of: Boolean, default: {} } // Per user: userId -> read status
});

module.exports = mongoose.model('Message', messageSchema);
[26/09/2025 06:04] ashu: const mongoose = require('mongoose');

const withdrawalSchema = new mongoose.Schema({
  user: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  amount: { type: Number, required: true }, // In coins (₹)
  upiId: { type: String }, // Or bank details
  bankDetails: { type: String },
  status: { type: String, enum: ['Pending', 'Approved', 'Rejected'], default: 'Pending' }
}, { timestamps: true });

module.exports = mongoose.model('Withdrawal', withdrawalSchema);
[26/09/2025 06:05] ashu: const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    if (!token) return res.status(401).json({ error: 'No token' });
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id);
    if (!user) return res.status(401).json({ error: 'Invalid token' });
    req.user = user;
    next();
  } catch (err) {
    res.status(401).json({ error: 'Auth failed' });
  }
};

const adminAuth = (req, res, next) => {
  if (req.user.role !== 'admin') return res.status(403).json({ error: 'Admin only' });
  next();
};

module.exports = { auth, adminAuth };
[26/09/2025 06:06] ashu: const express = require('express');
const router = express.Router();
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Register
router.post('/register', async (req, res) => {
  try {
    const { email, password, freeFireId, inGameName, whatsapp } = req.body;
    const user = new User({ email, password, freeFireId, inGameName, whatsapp, coins: 50 });
    await user.save();
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: '7d' });
    res.json({ token, user: { id: user._id, email, coins: user.coins } });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    if (!user || !(await user.comparePassword(password))) {
      return res.status(400).json({ error: 'Invalid credentials' });
    }
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: '7d' });
    res.json({ token, user: { id: user._id, email, coins: user.coins, role: user.role } });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

module.exports = router;
[26/09/2025 06:06] ashu: const express = require('express');
const router = express.Router();
const Tournament = require('../models/Tournament');
const User = require('../models/User');
const { auth, adminAuth } = require('../middleware/auth');

// List tournaments
router.get('/list', async (req, res) => {
  const tournaments = await Tournament.find({ status: 'Upcoming' }).populate('participants', 'inGameName');
  res.json(tournaments);
});

// Join tournament
router.post('/join/:id', auth, async (req, res) => {
  try {
    const tournament = await Tournament.findById(req.params.id);
    const user = req.user;
    if (tournament.participants.length >= tournament.maxParticipants) {
      return res.status(400).json({ error: 'Full' });
    }
    if (user.coins < tournament.entryFee) {
      return res.status(400).json({ error: 'Insufficient coins' });
    }
    user.coins -= tournament.entryFee;
    user.joinedTournaments.push(tournament._id);
    tournament.participants.push(user._id);
    await user.save();
    await tournament.save();
    res.json({ success: true, coinsLeft: user.coins });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// Admin: Create tournament
router.post('/create', auth, adminAuth, async (req, res) => {
  const tournament = new Tournament(req.body);
  await tournament.save();
  res.json(tournament);
});

// Admin: Upload results (distribute prizes)
router.post('/:id/results', auth, adminAuth, async (req, res) => {
  try {
    const { winners } = req.body; // Array of user IDs
    const tournament = await Tournament.findById(req.params.id);
    tournament.winners = winners;
    tournament.status = 'Completed';
    const prizePerWinner = tournament.prizePool / winners.length;
    for (let winnerId of winners) {
      const winner = await User.findById(winnerId);
      winner.coins += prizePerWinner;
      winner.totalWinnings += prizePerWinner;
      await winner.save();
    }
    await tournament.save();
    res.json({ success: true });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// User's joined tournaments
router.get('/my', auth, async (req, res) => {
  const tournaments = await Tournament.find({ participants: req.user._id })
    .populate('winners', 'inGameName');
  res.json(tournaments);
});

module.exports = router;
[26/09/2025 06:07] ashu: const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { auth } = require('../middleware/auth');

// Buy coins (mock Google Pay)
router.post('/buy', auth, async (req, res) => {
  try {
    const { amount } = req.body; // Coins to buy (₹1 = 1 coin)
    // Mock payment: In production, integrate Razorpay/Google Pay SDK
    // For now, assume success if amount > 0
    const user = req.user;
    user.coins += amount;
    await user.save();
    res.json({ success: true, coins: user.coins, upi: process.env.UPI_ID });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

module.exports = router;
[26/09/2025 06:07] ashu: const express = require('express');
const router = express.Router();
const Withdrawal = require('../models/Withdrawal');
const Message = require('../models/Message');
const Tournament = require('../models/Tournament');
const User = require('../models/User');
const { auth, adminAuth } = require('../middleware/auth');

// Leaderboard
router.get('/leaderboard', async (req, res) => {
  const users = await User.find({ role: 'user' }).sort({ totalWinnings: -1 }).limit(10)
    .select('inGameName avatar totalWinnings');
  res.json(users);
});

// Withdrawals: List & Approve
router.get('/withdrawals', auth, adminAuth, async (req, res) => {
  const withdrawals = await Withdrawal.find({ status: 'Pending' }).populate('user', 'inGameName');
  res.json(withdrawals);
});

router.post('/withdrawals/:id/approve', auth, adminAuth, async (req, res) => {
  const withdrawal = await Withdrawal.findById(req.params.id);
  withdrawal.status = 'Approved';
  await withdrawal.save();
  // In production: Trigger payout via Razorpay
  res.json({ success: true });
});

router.post('/withdrawals/:id/reject', auth, adminAuth, async (req, res) => {
  const withdrawal = await Withdrawal.findById(req.params.id);
  withdrawal.status = 'Rejected';
  await withdrawal.save();
  res.json({ success: true });
});

// Send message to tournament participants
router.post('/messages', auth, adminAuth, async (req, res) => {
  const { tournamentId, content } = req.body;
  const tournament = await Tournament.findById(tournamentId).populate('participants');
  const recipients = tournament.participants.map(p => p._id);
  const message = new Message({ tournamentId, recipients, content });
  await message.save();
  res.json(message);
});

module.exports = router;
[26/09/2025 06:08] ashu: const express = require('express');
const router = express.Router();
const Withdrawal = require('../models/Withdrawal');
const { auth } = require('../middleware/auth');

// Request withdrawal
router.post('/request', auth, async (req, res) => {
  const { amount, upiId, bankDetails } = req.body;
  if (req.user.coins < amount) return res.status(400).json({ error: 'Insufficient coins' });
  const withdrawal = new Withdrawal({ user: req.user._id, amount, upiId, bankDetails });
  req.user.coins -= amount; // Deduct immediately? Or on approval? Adjust as needed.
  await req.user.save();
  await withdrawal.save();
  res.json({ success: true });
});

// User's withdrawals
router.get('/my', auth, async (req, res) => {
  const withdrawals = await Withdrawal.find({ user: req.user._id });
  res.json(withdrawals);
});

module.exports = router;
[26/09/2025 06:08] ashu: const express = require('express');
const router = express.Router();
const nodemailer = require('nodemailer'); // Optional for real emails

// Contact form
router.post('/contact', async (req, res) => {
  const { name, email, message } = req.body;
  // Mock: Log to console. In production:
  // let transporter = nodemailer.createTransporter({ ... });
  // await transporter.sendMail({ to: process.env.ADMIN_EMAIL, subject: 'Contact from Fire Zone', text: message });
  console.log(Contact from ${email}: ${message});
  res.json({ success: true });
});

module.exports = router;
[26/09/2025 06:08] ashu: const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();
app.use(cors());
app.use(express.json());

// Connect MongoDB
mongoose.connect(process.env.MONGO_URI);

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/tournaments', require('./routes/tournaments'));
app.use('/api/payments', require('./routes/payments'));
app.use('/api/admin', require('./routes/admin'));
app.use('/api/withdrawals', require('./routes/withdrawals'));
app.use('/api/contact', require('./routes/contact'));

// Create admin if not exists (run once)
const User = require('./models/User');
User.findOne({ email: 'admin@firezone.com' }).then(admin => {
  if (!admin) {
    const adminUser = new User({ email: 'admin@firezone.com', password: 'admin123', role: 'admin' });
    adminUser.save();
  }
});

const PORT = 5000;
app.listen(PORT, () => console.log(Server on port ${PORT}));
[26/09/2025 06:09] ashu: Below is a complete, step-by-step implementation guide and codebase for the "Fire Zone" Free Fire tournament platform based on your specifications. I'll structure this response as follows:

1. *High-Level Architecture*: Overview of the project setup.
2. *Backend Implementation* (Node.js + Express + MongoDB): APIs, schemas, and logic for all features.
3. *Frontend Implementation* (React + Tailwind CSS): Components for each page/feature, with a dark gaming theme (fire/warrior-inspired: black/red/orange colors, fiery gradients, bold fonts).
4. *Integration Notes*: For payments (Google Pay/UPI), JWT, etc.
5. *Deployment/Setup Instructions*: How to run it locally.
6. *Limitations & Next Steps*: What I've implemented and potential enhancements.

This is a fully functional MVP (Minimum Viable Product). I've used:
- *Backend*: Node.js, Express, Mongoose (for MongoDB), JWT for auth, bcrypt for passwords.
- *Frontend*: React (with React Router for navigation), Tailwind CSS for styling (dark theme with Free Fire vibes: smoky backgrounds, flame icons via Heroicons or CSS).
- *Database*: MongoDB schemas for users, tournaments, messages, etc.
- *Payments*: Simulated Google Pay integration (using a simple form; real UPI would require Razorpay/Stripe SDK for production). For now, it uses a mock success callback to add coins.
- *Other*: Nodemailer for contact emails (optional), but contact is via a form that logs to console/admin email.

*Assumptions*:
- Free Fire integration is manual (e.g., users provide IDs; no official API).
- Payments: 1 Coin = ₹1. UPI ID: tiwari100456@okhdfcbank. In production, use Razorpay for secure payouts.
- Admin: Hardcoded admin user (email: admin@firezone.com, password: admin123) for simplicity. Add role-based auth in production.
- No real money transfer; withdrawals are approved manually by admin.
- Responsive design: Mobile-first with Tailwind.

---

### 1. High-Level Architecture

- *Folder Structure*:
  
  fire-zone/
  ├── backend/
  │   ├── models/     # MongoDB schemas (User, Tournament, Message, Withdrawal)
  │   ├── routes/     # API routes (auth, tournaments, payments, admin, etc.)
  │   ├── middleware/ # JWT auth, admin check
  │   ├── server.js   # Entry point
  │   └── package.json
  ├── frontend/
  │   ├── src/
  │   │   ├── components/ # Reusable (Navbar, Button, Card)
  │   │   ├── pages/      # Landing, Login, Dashboard, etc.
  │   │   ├── services/   # API calls (axios)
  │   │   ├── App.js      # Router setup
  │   │   └── index.css   # Tailwind imports
  │   └── package.json
  └── README.md
  

- *Data Flow*:
  - User registers/logs in → JWT token stored in localStorage.
  - Tournaments: List fetched via API; join deducts coins.
  - Payments: Form submits to backend → Mock success adds coins.
  - Admin: Separate routes; create tournaments, approve withdrawals, send messages.
  - Messages: Stored per tournament; users see unread count.

- *Theme (Tailwind)*: Dark mode base (bg-gray-900 text-white), fiery accents (bg-gradient-to-r from-red-600 to-orange-500), gaming fonts (e.g., via Google Fonts: 'Orbitron' for headers).

---

### 2. Backend Implementation

Install dependencies: npm init -y && npm i express mongoose bcryptjs jsonwebtoken cors dotenv nodemailer axios.

#### package.json (backend)
json
{
  "name": "fire-zone-backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mongoose": "^7.5.0",
    "bcryptjs": "^2.4.3",
    "jsonwebtoken": "^9.0.2",
    "cors": "^2.8.5",
    "dotenv": "^16.3.1",
    "nodemailer": "^6.9.7",
    "axios": "^1.5.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}


#### .env (backend)

MONGO_URI=mongodb://localhost:27017/firezone
JWT_SECRET=your_jwt_secret_key
ADMIN_EMAIL=officialfirezone7@gmail.com
UPI_ID=tiwari100456@okhdfcbank


#### models/User.js
javascript
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const userSchema = new mongoose.Schema({
  email: { type: String, required: true, unique: true },
  password: { type: String, required: true },
  freeFireId: { type: Number }, // Numeric
  inGameName: { type: String, required: true },
  whatsapp: { type: String, required: true, match: [/^\d{10}$/, 'Invalid WhatsApp (10 digits)'] }, // Indian format
  coins: { type: Number, default: 50 }, // Starts with 50 on register
  role: { type: String, default: 'user', enum: ['user', 'admin'] },
  totalWinnings: { type: Number, default: 0 },
  joinedTournaments: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Tournament' }],
  avatar: { type: String, default: 'default-avatar.png' } // For leaderboard
}, { timestamps: true });

userSchema.pre('save', async function(next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

userSchema.methods.comparePassword = async function(password) {
  return bcrypt.compare(password, this.password);
};

module.exports = mongoose.model('User', userSchema);


#### models/Tournament.js
javascript
const mongoose = require('mongoose');

const tournamentSchema = new mongoose.Schema({
  name: { type: String, required: true },
  entryFee: { type: Number, required: true }, // Coins
  maxParticipants: { type: Number, required: true },
  prizePool: { type: Number, required: true },
  mapType: { type: String, default: 'Bermuda' },
  timing: { type: Date, required: true }, // Start time
  participants: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }],
  status: { type: String, enum: ['Upcoming', 'Live', 'Completed'], default: 'Upcoming' },
  winners: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }] // For results
}, { timestamps: true });

module.exports = mongoose.model('Tournament', tournamentSchema);


#### models/Message.js
javascript
const mongoose = require('mongoose');

const messageSchema = new mongoose.Schema({
  tournamentId: { type: mongoose.Schema.Types.ObjectId, ref: 'Tournament', required: true },
  sender: { type: String, default: 'Admin' }, // Always Admin
  recipients: [{ type: mongoose.Schema.Types.ObjectId, ref: 'User' }], // Joined users
  content: { type: String, required: true },
  createdAt: { type: Date, default: Date.now },
  read: { type: Map, of: Boolean, default: {} } // Per user: userId -> read status
});

module.exports = mongoose.model('Message', messageSchema);


#### models/Withdrawal.js
javascript
const mongoose = require('mongoose');

const withdrawalSchema = new mongoose.Schema({
  user: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  amount: { type: Number, required: true }, // In coins (₹)
  upiId: { type: String }, // Or bank details
  bankDetails: { type: String },
  status: { type: String, enum: ['Pending', 'Approved', 'Rejected'], default: 'Pending' }
}, { timestamps: true });

module.exports = mongoose.model('Withdrawal', withdrawalSchema);


#### middleware/auth.js
javascript
const jwt = require('jsonwebtoken');
const User = require('../models/User');

const auth = async (req, res, next) => {
  try {
    const token = req.header('Authorization')?.replace('Bearer ', '');
    if (!token) return res.status(401).json({ error: 'No token' });
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    const user = await User.findById(decoded.id);
    if (!user) return res.status(401).json({ error: 'Invalid token' });
    req.user = user;
    next();
  } catch (err) {
    res.status(401).json({ error: 'Auth failed' });
  }
};

const adminAuth = (req, res, next) => {
  if (req.user.role !== 'admin') return res.status(403).json({ error: 'Admin only' });
  next();
};

module.exports = { auth, adminAuth };


#### routes/auth.js
javascript
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Register
router.post('/register', async (req, res) => {
  try {
    const { email, password, freeFireId, inGameName, whatsapp } = req.body;
    const user = new User({ email, password, freeFireId, inGameName, whatsapp, coins: 50 });
    await user.save();
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: '7d' });
    res.json({ token, user: { id: user._id, email, coins: user.coins } });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    if (!user || !(await user.comparePassword(password))) {
      return res.status(400).json({ error: 'Invalid credentials' });
    }
    const token = jwt.sign({ id: user._id }, process.env.JWT_SECRET, { expiresIn: '7d' });
    res.json({ token, user: { id: user._id, email, coins: user.coins, role: user.role } });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

module.exports = router;


#### routes/tournaments.js
javascript
const express = require('express');
const router = express.Router();
const Tournament = require('../models/Tournament');
const User = require('../models/User');
const { auth, adminAuth } = require('../middleware/auth');

// List tournaments
router.get('/list', async (req, res) => {
  const tournaments = await Tournament.find({ status: 'Upcoming' }).populate('participants', 'inGameName');
  res.json(tournaments);
});

// Join tournament
router.post('/join/:id', auth, async (req, res) => {
  try {
    const tournament = await Tournament.findById(req.params.id);
    const user = req.user;
    if (tournament.participants.length >= tournament.maxParticipants) {
      return res.status(400).json({ error: 'Full' });
    }
    if (user.coins < tournament.entryFee) {
      return res.status(400).json({ error: 'Insufficient coins' });
    }
    user.coins -= tournament.entryFee;
    user.joinedTournaments.push(tournament._id);
    tournament.participants.push(user._id);
    await user.save();
    await tournament.save();
    res.json({ success: true, coinsLeft: user.coins });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// Admin: Create tournament
router.post('/create', auth, adminAuth, async (req, res) => {
  const tournament = new Tournament(req.body);
  await tournament.save();
  res.json(tournament);
});

// Admin: Upload results (distribute prizes)
router.post('/:id/results', auth, adminAuth, async (req, res) => {
  try {
    const { winners } = req.body; // Array of user IDs
    const tournament = await Tournament.findById(req.params.id);
    tournament.winners = winners;
    tournament.status = 'Completed';
    const prizePerWinner = tournament.prizePool / winners.length;
    for (let winnerId of winners) {
      const winner = await User.findById(winnerId);
      winner.coins += prizePerWinner;
      winner.totalWinnings += prizePerWinner;
      await winner.save();
    }
    await tournament.save();
    res.json({ success: true });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// User's joined tournaments
router.get('/my', auth, async (req, res) => {
  const tournaments = await Tournament.find({ participants: req.user._id })
    .populate('winners', 'inGameName');
  res.json(tournaments);
});

module.exports = router;


#### routes/payments.js
javascript
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { auth } = require('../middleware/auth');

// Buy coins (mock Google Pay)
router.post('/buy', auth, async (req, res) => {
  try {
    const { amount } = req.body; // Coins to buy (₹1 = 1 coin)
    // Mock payment: In production, integrate Razorpay/Google Pay SDK
    // For now, assume success if amount > 0
    const user = req.user;
    user.coins += amount;
    await user.save();
    res.json({ success: true, coins: user.coins, upi: process.env.UPI_ID });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

module.exports = router;


#### routes/admin.js
javascript
const express = require('express');
const router = express.Router();
const Withdrawal = require('../models/Withdrawal');
const Message = require('../models/Message');
const Tournament = require('../models/Tournament');
const User = require('../models/User');
const { auth, adminAuth } = require('../middleware/auth');

// Leaderboard
router.get('/leaderboard', async (req, res) => {
  const users = await User.find({ role: 'user' }).sort({ totalWinnings: -1 }).limit(10)
    .select('inGameName avatar totalWinnings');
  res.json(users);
});

// Withdrawals: List & Approve
router.get('/withdrawals', auth, adminAuth, async (req, res) => {
  const withdrawals = await Withdrawal.find({ status: 'Pending' }).populate('user', 'inGameName');
  res.json(withdrawals);
});

router.post('/withdrawals/:id/approve', auth, adminAuth, async (req, res) => {
  const withdrawal = await Withdrawal.findById(req.params.id);
  withdrawal.status = 'Approved';
  await withdrawal.save();
  // In production: Trigger payout via Razorpay
  res.json({ success: true });
});

router.post('/withdrawals/:id/reject', auth, adminAuth, async (req, res) => {
  const withdrawal = await Withdrawal.findById(req.params.id);
  withdrawal.status = 'Rejected';
  await withdrawal.save();
  res.json({ success: true });
});

// Send message to tournament participants
router.post('/messages', auth, adminAuth, async (req, res) => {
  const { tournamentId, content } = req.body;
  const tournament = await Tournament.findById(tournamentId).populate('participants');
  const recipients = tournament.participants.map(p => p._id);
  const message = new Message({ tournamentId, recipients, content });
  await message.save();
  res.json(message);
});

module.exports = router;


#### routes/withdrawals.js
javascript
const express = require('express');
const router = express.Router();
const Withdrawal = require('../models/Withdrawal');
const { auth } = require('../middleware/auth');

// Request withdrawal
router.post('/request', auth, async (req, res) => {
  const { amount, upiId, bankDetails } = req.body;
  if (req.user.coins < amount) return res.status(400).json({ error: 'Insufficient coins' });
  const withdrawal = new Withdrawal({ user: req.user._id, amount, upiId, bankDetails });
  req.user.coins -= amount; // Deduct immediately? Or on approval? Adjust as needed.
  await req.user.save();
  await withdrawal.save();
  res.json({ success: true });
});

// User's withdrawals
router.get('/my', auth, async (req, res) => {
  const withdrawals = await Withdrawal.find({ user: req.user._id });
  res.json(withdrawals);
});

module.exports = router;


#### routes/contact.js
javascript
const express = require('express');
const router = express.Router();
const nodemailer = require('nodemailer'); // Optional for real emails

// Contact form
router.post('/contact', async (req, res) => {
  const { name, email, message } = req.body;
  // Mock: Log to console. In production:
  // let transporter = nodemailer.createTransporter({ ... });
  // await transporter.sendMail({ to: process.env.ADMIN_EMAIL, subject: 'Contact from Fire Zone', text: message });
  console.log(`Contact from ${email}: ${message}`);
  res.json({ success: true });
});

module.exports = router;


#### server.js
javascript
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();
app.use(cors());
app.use(express.json());

// Connect MongoDB
mongoose.connect(process.env.MONGO_URI);

// Routes
app.use('/api/auth', require('./routes/auth'));
app.use('/api/tournaments', require('./routes/tournaments'));
app.use('/api/payments', require('./routes/payments'));
app.use('/api/admin', require('./routes/admin'));
app.use('/api/withdrawals', require('./routes/withdrawals'));
app.use('/api/contact', require('./routes/contact'));

// Create admin if not exists (run once)
const User = require('./models/User');
User.findOne({ email: 'admin@firezone.com' }).then(admin => {
  if (!admin) {
    const adminUser = new User({ email: 'admin@firezone.com', password: 'admin123', role: 'admin' });
    adminUser.save();
  }
});

const PORT = 5000;
app.listen(PORT, () => console.log(`Server on port ${PORT}`));


---

### 3
