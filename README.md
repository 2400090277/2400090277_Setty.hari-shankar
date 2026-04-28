const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");
const bodyParser = require("body-parser");
const jwt = require("jsonwebtoken");
const bcrypt = require("bcryptjs");

const app = express();
app.use(cors());
app.use(bodyParser.json());

/* ================= DB ================= */
mongoose.connect("mongodb://127.0.0.1:27017/erp");

/* ================= MODELS ================= */
const User = mongoose.model("User", {
  name: String,
  email: String,
  password: String,
  role: String,
});

const Student = mongoose.model("Student", {
  name: String,
  course: String,
  attendance: Number,
  grades: Number,
});

/* ================= MIDDLEWARE ================= */
const auth = (req, res, next) => {
  const token = req.headers.authorization;
  if (!token) return res.send("No token");

  try {
    const decoded = jwt.verify(token, "secret");
    req.user = decoded;
    next();
  } catch {
    res.send("Invalid token");
  }
};

/* ================= AUTH ================= */
app.post("/register", async (req, res) => {
  const { name, email, password, role } = req.body;

  const hash = await bcrypt.hash(password, 10);

  const user = await User.create({
    name,
    email,
    password: hash,
    role,
  });

  res.send(user);
});

app.post("/login", async (req, res) => {
  const { email, password } = req.body;

  const user = await User.findOne({ email });
  if (!user) return res.send("User not found");

  const match = await bcrypt.compare(password, user.password);
  if (!match) return res.send("Wrong password");

  const token = jwt.sign(
    { id: user._id, role: user.role },
    "secret"
  );

  res.send({ token, user });
});

/* ================= STUDENTS ================= */
app.get("/students", auth, async (req, res) => {
  const data = await Student.find();
  res.send(data);
});

app.post("/students", auth, async (req, res) => {
  if (req.user.role !== "teacher" && req.user.role !== "admin") {
    return res.send("Access denied");
  }

  const student = await Student.create(req.body);
  res.send(student);
});

/* ================= FRONTEND ================= */
app.get("/", (req, res) => {
  res.send(`
  <html>
  <h2>ERP SYSTEM</h2>

  <h3>Register</h3>
  <input id="rname" placeholder="Name"/><br/>
  <input id="remail" placeholder="Email"/><br/>
  <input id="rpass" placeholder="Password"/><br/>
  <select id="rrole">
    <option>admin</option>
    <option>teacher</option>
    <option>student</option>
  </select><br/>
  <button onclick="register()">Register</button>

  <h3>Login</h3>
  <input id="lemail" placeholder="Email"/><br/>
  <input id="lpass" placeholder="Password"/><br/>
  <button onclick="login()">Login</button>

  <h3>Add Student</h3>
  <input id="sname" placeholder="Name"/><br/>
  <input id="scourse" placeholder="Course"/><br/>
  <input id="sattendance" placeholder="Attendance"/><br/>
  <input id="sgrades" placeholder="Grades"/><br/>
  <button onclick="addStudent()">Add</button>

  <h3>Students List</h3>
  <button onclick="getStudents()">Load Students</button>
  <ul id="list"></ul>

<script>
let token = "";

async function register() {
  await fetch("/register", {
    method:"POST",
    headers:{ "Content-Type":"application/json" },
    body:JSON.stringify({
      name: rname.value,
      email: remail.value,
      password: rpass.value,
      role: rrole.value
    })
  });
  alert("Registered");
}

async function login() {
  const res = await fetch("/login", {
    method:"POST",
    headers:{ "Content-Type":"application/json" },
    body:JSON.stringify({
      email: lemail.value,
      password: lpass.value
    })
  });
  const data = await res.json();
  token = data.token;
  alert("Logged in");
}

async function addStudent() {
  await fetch("/students", {
    method:"POST",
    headers:{
      "Content-Type":"application/json",
      "Authorization": token
    },
    body:JSON.stringify({
      name: sname.value,
      course: scourse.value,
      attendance: sattendance.value,
      grades: sgrades.value
    })
  });
  alert("Student added");
}

async function getStudents() {
  const res = await fetch("/students", {
    headers:{ "Authorization": token }
  });
  const data = await res.json();

  list.innerHTML = "";
  data.forEach(s => {
    list.innerHTML += "<li>" + s.name + " - " + s.course + "</li>";
  });
}
</script>
  </html>
  `);
});

/* ================= SERVER ================= */
app.listen(5000, () => {
  console.log("Server running on http://localhost:5000");
});
