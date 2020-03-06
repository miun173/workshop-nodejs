# Workshop Unida
## Start
Kita akan pakai `repository` pattern, jadi semua akses ke storage/thirdparty 
akan dilakukan di modul repository.
```
.
├── config.js
├── db
│   ├── migrations.js
│   ├── mysql.js
│   └── utils.js
├── main.js
├── package.json
├── package-lock.json
├── README.md
├── repository
│   └── user_repository.js
└── service
    ├── service.js
```

Buat folder project lalu buka terminal dan ketikkan
```
npm init -y
```

Lalu, kita install dependesi berikut
```
npm i express body-parser mysql nodemon
```

Buat service `service/service.js`
```js
const express = require('express')
const bodyParser = require('body-parser')

const app = express()

// create application/json parser
app.use(bodyParser.json({ type: 'application/json' }))
app.get("/ping", (req, res) => res.json({"ping": "pong"}))

exports.start = () => {
    app.listen(3000, () => {
        console.log("server start @ :3000")
    })
}

module.exports = exports
```

Kita buat file `main.js` di root folder
```js
const service = require('./service/service.js')

service.start()
```

Kita tes
```
curl localhost:3000/ping
```

## Features
1. Sebagai pengunjung saya ingin dapat mendaftarkan diri ke platform, menggunakan username dan password
2. Sebagai pemilik platform berita hanya dapat ditulis/diubah/dihapus oleh pengguna yg terotorisasi
3. Sebagai pengunjung saya ingin melihat daftar berita yang berisi judul, penulis, tanggal terbit dan isi berita


## Design
```
User {
    id: Int
    username: String
    password: String
    name: String
}

Content {
    id: Int
    author_id: Int
    title: String
    body: String
}
```

### Feature 1:
    - register --> check user exists
        - not exist --> create user
        - exists --> error 
    
    - login --> check user exists
        - not exists --> error
        - exists --> validate password 
            - invalid password --> error
            - valid --> logged in


## User service
Kita akan buat `user service` ini adalah salah satu core karena akan digunakan untuk
per-auth-an. Tapi sebelum nya kita akan setup database dulu. 
Buat file `db/mysql.js`
```js
const mysql = require('mysql');
let pool;
module.exports = {
    /**
     * @returns {mysql.Pool}
     */
    getPool: function () {
      if (pool) return pool;

      pool = mysql.createPool({
        host: '127.0.0.1',
        port: 3306,
        user: 'root',
        password: 'root',
        database: 'workshop'
      });

      return pool;
    }
};

```

Lalu kita buat `database migration`, kita bikin `utils` untuk jalanin migration.
Buat file `db/utils.js`
```js
// db/utils.js
const db = require('./mysql')
const pool = db.getPool()

exports.migrate = (queries = []) => {
    pool.getConnection((err, conn) => {
        if (err) {
            console.error(err)
            return
        }
    
        queries.forEach(q => {
            conn.query(q, (err, result) => {
                if (err) {
                    console.error(err)

                    conn.release()
                    break
                }
            })
        })

        conn.release()
        pool.end()
    })
}

module.exports = exports
```

Kita akan tulis migration2 nya di file `db/migrations.js`, kita perlu bikin user migration
```js
const utils = require('./utils.js')
let createTableUser = `
    CREATE TABLE IF NOT EXISTS users (
        id INT NOT NULL AUTO_INCREMENT,
        username VARCHAR(255),
        name VARCHAR(255),
        password TEXT,
        CONSTRAINT users_username_unique UNIQUE (username),
        PRIMARY KEY (id)
    );
`

utils.migrate([createTableUser])
```

sip, sebelum kita jalanin migrasi nya, nyalain mysql dan bikin database dengan nama `workshop`.
```
node db/migration.js
```

Sekarang kita buat `user_repository`, di sini lah fungsi CRUD dibuat. Untuk fitur satu, kita perlu 
fungsi `create user`
```js
const db = require('../db/mysql.js')
const pool = db.getPool()

exports.create = ({ username = '', name = '', password = '' }) => {
    return new Promise((resolve, reject) => {
        pool.getConnection((err, conn) => {
            if (err) {
                console.error(err)
                return
            }
            
            const q = `INSERT INTO users (username, name, password) VALUES(?, ?, ?)`
            conn.query(q, [username, name, password], (err, res) => {
                conn.release()

                if (err) {
                    console.error(err)
                    reject(err)
                    return
                }

                let id = results.insertId
                resolve({ username, name, id })
            })
        })
    })
}
```

Kita bisa test langsung buat file `test.js`
```js
const userRepo = require('./repository/user_repository.js')

userRepo.create({ username: "miun173", name: "fahmi", password: "test" })
    .then(user => {
        console.log(user)
    })
    .catch(err => {
        console.error(err)
    })
    .finally(() => {
        process.exit(0)
    })
```
