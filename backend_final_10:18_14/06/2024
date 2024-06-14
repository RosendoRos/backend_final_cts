const express = require('express');
const mysql = require('mysql2');
const bodyParser = require('body-parser');
const ExcelJS = require('exceljs');
const moment = require('moment-timezone');
const cron = require('node-cron');
const path = require('path'); 
const fs = require('fs');
const TIMEZONE = 'America/Mexico_City';

const app = express();
const port = 5000;

// Middleware
app.use(bodyParser.json());

// MySQL connection
const db = mysql.createConnection({
  host: 'localhost',
  user: 'root',
  password: '',
  database: 'ctsinge'
});

db.connect(err => {
  if (err) {
    console.error('Error connecting to MySQL:', err);
    return;
  }
  console.log('Connected to MySQL');
});

// Route to register a new user
app.post('/api/register', (req, res) => {
  const { username, password } = req.body;

  const checkUserQuery = 'SELECT * FROM users WHERE username = ?';
  db.query(checkUserQuery, [username], (err, results) => {
    if (err) {
      console.error('Error checking user existence:', err);
      res.status(500).send('Failed to check user existence');
      return;
    }
    if (results.length > 0) {
      res.status(400).send('Username already taken');
      return;
    }

    const query = 'INSERT INTO users (username, password) VALUES (?, ?)';
    db.query(query, [username, password], (err, results) => {
      if (err) {
        console.error('Error registering user:', err);
        res.status(500).send('Failed to register user');
        return;
      }
      res.status(200).send('User registered successfully');
    });
  });
});

// Route to login a user
app.post('/api/login', (req, res) => {
  const { username, password } = req.body;

  const query = 'SELECT * FROM users WHERE username = ? AND password = ?';
  db.query(query, [username, password], (err, results) => {
    if (err) {
      console.error('Error logging in user:', err);
      res.status(500).send('Failed to login');
      return;
    }
    if (results.length > 0) {
      const user = results[0];
      if (user.username === '1') {
        res.status(200).send({ message: 'Login successful', isAdmin: true });
      } else {
        res.status(200).send({ message: 'Login successful', isAdmin: false });
      }
    } else {
      res.status(401).send({ message: 'Invalid username or password' });
    }
  });
});

const calculateAndUpdateWeeklyHours = async () => {
  const startOfWeek = moment().tz(TIMEZONE).startOf('week').toDate();
  const endOfWeek = moment().tz(TIMEZONE).endOf('week').toDate();

  const users = await new Promise((resolve, reject) => {
    db.query('SELECT DISTINCT id_unico FROM scans', (err, results) => {
      if (err) reject(err);
      else resolve(results.map(result => result.id_unico));
    });
  });

  for (const userId of users) {
    const userEntries = await new Promise((resolve, reject) => {
      db.query('SELECT * FROM scans WHERE id_unico = ? AND timestamp >= ? AND timestamp <= ?', [userId, startOfWeek, endOfWeek], (err, results) => {
        if (err) reject(err);
        else resolve(results);
      });
    });

    let totalMinutes = 0;
    let lastEntry = null;

    userEntries.forEach((entry) => {
      if (entry.entrada_sali === 'entrada') {
        lastEntry = moment.tz(entry.timestamp, TIMEZONE);
      } else if (entry.entrada_sali === 'salida' && lastEntry) {
        const endTime = moment.tz(entry.timestamp, TIMEZONE);
        const duration = moment.duration(endTime.diff(lastEntry));
        totalMinutes += duration.asMinutes();
        lastEntry = null;
      }
    });

    const totalHours = totalMinutes / 60;

    await new Promise((resolve, reject) => {
      db.query('UPDATE scans SET total_hours = ? WHERE id_unico = ? AND timestamp >= ? AND timestamp <= ?', [totalHours, userId, startOfWeek, endOfWeek], (err, results) => {
        if (err) reject(err);
        else resolve(results);
      });
    });
  }
};

app.post('/api/save-scan', (req, res) => {
  const { name, puesto, timestamp, entrada_sali, location, id_unico } = req.body;
  const { latitude, longitude } = location || {};

  // Verificar si ya existe un registro de entrada o salida para el usuario en la fecha actual
  const date = moment(timestamp).format('YYYY-MM-DD');
  const checkQuery = `
    SELECT * FROM scans 
    WHERE id_unico = ? 
      AND DATE(timestamp) = ? 
      AND entrada_sali = ?
  `;
  db.query(checkQuery, [id_unico, date, entrada_sali], (err, results) => {
    if (err) {
      console.error('Error checking scan data:', err);
      res.status(500).send('Failed to check scan data');
      return;
    }
    if (results.length > 0) {
      res.status(400).send(`Ya has registrado una ${entrada_sali} hoy`);
      return;
    }

    // Insertar nuevo registro de escaneo
    const query = 'INSERT INTO scans (name, puesto, timestamp, entrada_sali, latitude, longitude, id_unico) VALUES (?, ?, ?, ?, ?, ?, ?)';
    db.query(query, [name, puesto, timestamp, entrada_sali, latitude, longitude, id_unico], async (err, results) => {
      if (err) {
        console.error('Error inserting scan data:', err);
        res.status(500).send('Failed to save scan data');
        return;
      }
      
      try {
        // Actualizar horas totales para la semana actual del usuario
        await calculateAndUpdateWeeklyHours(id_unico);
        res.status(200).send('Scan data saved successfully and total hours updated');
      } catch (updateErr) {
        console.error('Error updating total hours:', updateErr);
        res.status(500).send('Scan data saved but failed to update total hours');
      }
    });
  });
});

// Route to get all unique users from scans
app.get('/api/users', (req, res) => {
  const startOfWeek = moment().startOf('week').format('YYYY-MM-DD HH:mm:ss');
  const endOfWeek = moment().endOf('week').format('YYYY-MM-DD HH:mm:ss');

  const query = 'SELECT DISTINCT id_unico, name FROM scans';
  db.query(query, (err, results) => {
    if (err) {
      console.error('Error fetching users:', err);
      res.status(500).send('Failed to fetch users');
      return;
    }
    res.status(200).json(results);
  });
});

// Route to get details of a specific user
app.get('/api/users/:id_unico', (req, res) => {
  const { id_unico } = req.params;

  const startOfWeek = moment().startOf('week').format('YYYY-MM-DD HH:mm:ss');
  const endOfWeek = moment().endOf('week').format('YYYY-MM-DD HH:mm:ss');

  const query = 'SELECT * FROM scans WHERE id_unico = ? AND timestamp BETWEEN ? AND ? ORDER BY timestamp DESC';
  db.query(query, [id_unico, startOfWeek, endOfWeek], (err, results) => {
    if (err) {
      console.error('Error fetching user details:', err);
      res.status(500).send('Failed to fetch user details');
      return;
    }
    res.status(200).json(results);
  });
});

// Route to get all scans (for administrators)
app.get('/api/scans', (req, res) => {
  const query = 'SELECT * FROM scans';
  db.query(query, (err, results) => {
    if (err) {
      console.error('Error fetching scans:', err);
      res.status(500).send('Failed to fetch scans');
      return;
    }
    res.status(200).json(results);
  });
});

// New Route to calculate and add extra hours
app.post('/api/add-extra-hours', (req, res) => {
  const { id_unico, extra_hours } = req.body;

  const query = 'UPDATE users SET extra_hours = ? WHERE id_unico = ?';
  db.query(query, [extra_hours, id_unico], (err, results) => {
    if (err) {
      console.error('Error updating extra hours:', err);
      res.status(500).send('Failed to update extra hours');
      return;
    }
    res.status(200).send('Extra hours added successfully');
  });
});

app.get('/api/generate-excel-report', async (req, res) => {
  try {
    // Actualizar las horas totales antes de generar el informe
    await calculateAndUpdateWeeklyHours();

    const startOfWeek = moment().tz(TIMEZONE).startOf('week').toDate();
    const endOfWeek = moment().tz(TIMEZONE).endOf('week').toDate();

    const rows = await new Promise((resolve, reject) => {
      db.query('SELECT * FROM scans WHERE timestamp >= ? AND timestamp <= ?', [startOfWeek, endOfWeek], (err, results) => {
        if (err) reject(err);
        else resolve(results);
      });
    });

    if (!Array.isArray(rows) || rows.length === 0) {
      throw new Error('No data found for the current week');
    }

    const workbook = new ExcelJS.Workbook();
    const worksheet = workbook.addWorksheet('Weekly Report');

    worksheet.columns = [
      { header: 'ID', key: 'id_unico', width: 10 },
      { header: 'Nombre', key: 'name', width: 30 },
      { header: 'Puesto', key: 'puesto', width: 30 },
      { header: 'Fecha y Hora', key: 'timestamp', width: 30 },
      { header: 'Acción', key: 'entrada_sali', width: 15 },
      { header: 'Latitud', key: 'latitude', width: 15 },
      { header: 'Longitud', key: 'longitude', width: 15 },
      { header: 'Horas Extras', key: 'horas_extras', width: 15 },
      { header: 'Total Horas', key: 'total_hours', width: 15 }
    ];

    rows.forEach((row) => {
      worksheet.addRow({
        id_unico: row.id_unico,
        name: row.name,
        puesto: row.puesto,
        timestamp: moment.tz(row.timestamp, TIMEZONE).format('YYYY-MM-DD HH:mm:ss'),
        entrada_sali: row.entrada_sali,
        latitude: row.latitude,
        longitude: row.longitude,
        horas_extras: row.horas_extras || 0,
        total_hours: row.total_hours || 0
      });
    });

    res.setHeader('Content-Type', 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet');
    res.setHeader('Content-Disposition', 'attachment; filename=weekly_report.xlsx');

    await workbook.xlsx.write(res);
    res.end();
  } catch (error) {
    res.status(500).send({ error: error.message });
  }
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
