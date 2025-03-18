const express = require('express');
const mysql = require('mysql');
const bodyParser = require('body-parser');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(bodyParser.json());

const db = mysql.createConnection({
    host: 'sql206.infinityfree.com',       // MySQL Host Name
    user: 'if0_38006942',                  // MySQL User Name
    password: 'your_vpanel_password',      // Το password του vPanel (το ίδιο που χρησιμοποιείς για login)
    database: 'if0_38006942_myapp_db',     // MySQL DB Name
    charset: 'utf8mb4'                     // Για υποστήριξη ελληνικών χαρακτήρων
  });

db.connect(err => {
  if (err) throw err;
  console.log('Connected to database');
});

app.post('/api/submit', (req, res) => {
    const { firstName, lastName, thl, perioxh, diefthinsi, koudouni, sxolia, cartItems } = req.body;
    const userIp = req.ip;
    const userAgent = req.headers['user-agent'];
    const deviceHash = `${userIp}-${userAgent}`; // Συνδυασμός IP + User-Agent

    // Έλεγχος αν αυτή η συσκευή έχει κάνει παραγγελία το τελευταίο 1 λεπτό
    const checkTimeSQL = 'SELECT last_order_time FROM users WHERE device_hash = ? ORDER BY last_order_time DESC LIMIT 1';
    
    db.query(checkTimeSQL, [deviceHash], (err, results) => {
        if (err) {
            console.error(err);
            return res.status(500).send({ message: 'Database error' });
        }

        if (results.length > 0) {
            const lastOrderTime = new Date(results[0].last_order_time);
            const currentTime = new Date();
            const timeDiff = (currentTime - lastOrderTime) / (1000 * 60); // Διαφορά σε λεπτά

            if (timeDiff < 1) { // Περιορισμός χρόνου
                return res.status(429).send({ message: 'Πρέπει να περιμένετε 1 λεπτό πριν ξαναπαραγγείλετε.' });
            }
        }

        // Εισαγωγή νέας παραγγελίας
        const sqlOrder = 'INSERT INTO users (first_name, last_name, thl, perioxh, diefthinsi, koudouni, sxolia, device_hash, last_order_time) VALUES (?, ?, ?, ?, ?, ?, ?, ?, NOW())';
        db.query(sqlOrder, [firstName, lastName, thl, perioxh, diefthinsi, koudouni, sxolia, deviceHash], (err, result) => {
            if (err) {
                console.error(err);
                return res.status(500).send({ message: 'Error saving order' });
            }

            const orderId = result.insertId;
            const sqlOrderItems = 'INSERT INTO users_order (order_id, name, price, quantity) VALUES ?';
            const orderItems = cartItems.map(item => [orderId, item.name, item.price, item.quantity]);

            db.query(sqlOrderItems, [orderItems], (err, result) => {
                if (err) {
                    console.error(err);
                    return res.status(500).send({ message: 'Error saving order items' });
                }

                res.status(200).send({ message: 'Order placed successfully!' });
            });
        });
    });
});

app.listen(5000, () => {
  console.log('Server running on port 5000');
});
