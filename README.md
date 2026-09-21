const express = require('express');
const path = require('path');
const cors = require('cors');
const sqlite3 = require('sqlite3').verbose();

const app = express();
const PORT = process.env.PORT || 3000;

app.use(cors());
app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

// Khởi tạo cơ sở dữ liệu (SQLite file-based database)
const db = new sqlite3.Database('./database.sqlite', (err) => {
  if (err) {
    console.error('Lỗi kết nối CSDL:', err.message);
  } else {
    console.log('Đã kết nối thành công đến cơ sở dữ liệu SQLite.');
  }
});

// Tạo bảng dữ liệu nếu chưa tồn tại
db.serialize(() => {
  db.run(`
    CREATE TABLE IF NOT EXISTS transactions (
      id TEXT PRIMARY KEY,
      type TEXT NOT NULL,
      amount REAL NOT NULL,
      category TEXT NOT NULL,
      note TEXT,
      date TEXT NOT NULL
    )
  `);

  db.run(`
    CREATE TABLE IF NOT EXISTS savings_goals (
      id TEXT PRIMARY KEY,
      title TEXT NOT NULL,
      targetAmount REAL NOT NULL,
      currentAmount REAL DEFAULT 0
    )
  `);
});

// ==================== API TRANSACTIONS ====================

// Lấy danh sách giao dịch
app.get('/api/transactions', (req, res) => {
  db.all('SELECT * FROM transactions ORDER BY datetime(date) DESC', [], (err, rows) => {
    if (err) return res.status(500).json({ error: err.message });
    res.json(rows);
  });
});

// Thêm giao dịch mới
app.post('/api/transactions', (req, res) => {
  const { type, amount, category, note, date } = req.body;
  const id = Date.now().toString();
  const txDate = date || new Date().toISOString();

  const query = `INSERT INTO transactions (id, type, amount, category, note, date) VALUES (?, ?, ?, ?, ?, ?)`;
  db.run(query, [id, type, amount, category, note, txDate], function (err) {
    if (err) return res.status(500).json({ error: err.message });
    res.status(201).json({ id, type, amount, category, note, date: txDate });
  });
});

// Xóa giao dịch
app.delete('/api/transactions/:id', (req, res) => {
  db.run('DELETE FROM transactions WHERE id = ?', [req.params.id], function (err) {
    if (err) return res.status(500).json({ error: err.message });
    res.json({ message: 'Đã xóa giao dịch thành công' });
  });
});

// ==================== API SAVINGS GOALS ====================

// Lấy danh sách sổ tiết kiệm
app.get('/api/goals', (req, res) => {
  db.all('SELECT * FROM savings_goals', [], (err, rows) => {
    if (err) return res.status(500).json({ error: err.message });
    res.json(rows);
  });
});

// Tạo sổ tiết kiệm mới
app.post('/api/goals', (req, res) => {
  const { title, targetAmount } = req.body;
  const id = Date.now().toString();

  const query = `INSERT INTO savings_goals (id, title, targetAmount, currentAmount) VALUES (?, ?, ?, 0)`;
  db.run(query, [id, title, targetAmount], function (err) {
    if (err) return res.status(500).json({ error: err.message });
    res.status(201).json({ id, title, targetAmount, currentAmount: 0 });
  });
});

// Nạp hoặc rút tiền từ sổ
app.post('/api/goals/:id/transfer', (req, res) => {
  const { action, amount } = req.body; // action: 'deposit' | 'withdraw'
  const goalId = req.params.id;

  db.get('SELECT * FROM savings_goals WHERE id = ?', [goalId], (err, goal) => {
    if (err || !goal) return res.status(404).json({ error: 'Không tìm thấy sổ tiết kiệm' });

    let newAmount = goal.currentAmount;
    if (action === 'deposit') {
      newAmount += Number(amount);
    } else if (action === 'withdraw') {
      if (goal.currentAmount < amount) {
        return res.status(400).json({ error: 'Số dư sổ không đủ để rút!' });
      }
      newAmount -= Number(amount);
    }

    db.run('UPDATE savings_goals SET currentAmount = ? WHERE id = ?', [newAmount, goalId], function (updateErr) {
      if (updateErr) return res.status(500).json({ error: updateErr.message });

      // Ghi nhận biến động dòng tiền tương ứng
      const txId = Date.now().toString();
      const txType = action === 'deposit' ? 'expense' : 'income';
      const txCategory = action === 'deposit' ? 'Tiết kiệm' : 'Rút tiết kiệm';
      const txNote = action === 'deposit' ? `Nạp vào: ${goal.title}` : `Rút từ: ${goal.title}`;

      db.run(
        `INSERT INTO transactions (id, type, amount, category, note, date) VALUES (?, ?, ?, ?, ?, ?)`,
        [txId, txType, amount, txCategory, txNote, new Date().toISOString()]
      );

      res.json({ message: 'Cập nhật thành công', currentAmount: newAmount });
    });
  });
});

// Xóa sổ tiết kiệm
app.delete('/api/goals/:id', (req, res) => {
  db.run('DELETE FROM savings_goals WHERE id = ?', [req.params.id], function (err) {
    if (err) return res.status(500).json({ error: err.message });
    res.json({ message: 'Đã xóa sổ tiết kiệm' });
  });
});

// Serve frontend SPA fallback
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

app.listen(PORT, () => {
  console.log(`Server đang chạy tại cổng ${PORT}`);
});
