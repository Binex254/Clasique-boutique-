<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Clasique Boutique</title>
<style>
/* Global UI */
body{font-family:Arial;background:#f4f4f4;color:#333;margin:0;padding:0;}
header{background:#000;color:#fff;padding:15px;text-align:center;}
header nav a{color:#fff;margin:0 10px;text-decoration:none;font-weight:bold;}
header nav a:hover{color:#ffcc00;}
footer{background:#000;color:#fff;text-align:center;padding:15px;margin-top:20px;}
.container{max-width:1200px;margin:30px auto;padding:10px;}
button{background:#000;color:#fff;border:none;padding:10px;border-radius:4px;cursor:pointer;}
button:hover{background:#ffcc00;color:#000;}
input,select{width:100%;padding:10px;border:1px solid #ccc;border-radius:4px;margin:8px 0;}
.products-grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(200px,1fr));gap:20px;margin-top:20px;}
.product-card{background:#fff;padding:15px;border-radius:8px;box-shadow:0 4px 8px rgba(0,0,0,0.1);text-align:center;}
.product-card img{max-width:100%;border-radius:8px;}
.cart-item{display:flex;align-items:center;justify-content:space-between;padding:8px;background:#fff;margin-bottom:8px;border-radius:6px;box-shadow:0 2px 5px rgba(0,0,0,0.1);}
.cart-item img{width:50px;height:50px;border-radius:6px;margin-right:10px;}
#login,#register,#dashboard{max-width:400px;margin:auto;background:#fff;padding:20px;border-radius:8px;box-shadow:0 4px 8px rgba(0,0,0,0.1);}
</style>
</head>
<body>

<header>
<h1>Clasique Boutique</h1>
<nav>
<a href="#" onclick="showSection('login')">Login</a>
<a href="#" onclick="showSection('register')">Register</a>
<a href="#" onclick="showSection('dashboard')">Dashboard</a>
<a href="#" onclick="logout()">Logout</a>
</nav>
</header>

<!-- LOGIN -->
<div id="login" class="container">
<h2>Login</h2>
<input type="email" id="loginEmail" placeholder="Email">
<input type="password" id="loginPassword" placeholder="Password">
<button onclick="login()">Login</button>
</div>

<!-- REGISTER -->
<div id="register" class="container" style="display:none">
<h2>Register</h2>
<input type="email" id="registerEmail" placeholder="Email">
<input type="password" id="registerPassword" placeholder="Password">
<button onclick="register()">Register</button>
</div>

<!-- DASHBOARD -->
<div id="dashboard" class="container" style="display:none">
<h2 id="dashboardTitle">Dashboard</h2>

<!-- ADMIN SECTION -->
<div id="adminSection" style="display:none">
<h3>Add Product</h3>
<input type="text" id="productName" placeholder="Product Name">
<input type="number" id="productPrice" placeholder="Price (KES)">
<input type="file" id="productFile" accept="image/*">
<button onclick="addProduct()">Add Product</button>

<h3>All Products</h3>
<div id="productsContainer" class="products-grid"></div>
</div>

<!-- USER SECTION -->
<div id="userSection" style="display:none">
<h3>Products</h3>
<div id="productsContainerUser" class="products-grid"></div>

<h3>Shopping Cart</h3>
<div id="cartContainer"></div>
</div>

<!-- CONTACT INFO -->
<div style="margin-top:30px;text-align:center;">
<h3>Contact Us</h3>
<p>Phone: 0748617221</p>
<p>Email: <a href="mailto:gathunguadams@gmail.com">gathunguadams@gmail.com</a></p>
</div>

</div>
<footer>
© 2026 Clasique Boutique | 0748617221 | gathunguadams@gmail.com
</footer>

<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import { getAuth, createUserWithEmailAndPassword, signInWithEmailAndPassword, signOut } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js";
import { getFirestore, doc, setDoc, getDoc, collection, addDoc, getDocs, deleteDoc, updateDoc } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";
import { getStorage, ref, uploadBytes, getDownloadURL } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-storage.js";

// 🧠 Firebase Config — replace with YOUR keys
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_BUCKET.appspot.com",
  messagingSenderId: "YOUR_ID",
  appId: "YOUR_APP_ID"
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const storage = getStorage(app);

// ---------------- UI ----------------
function showSection(id){
  ["login","register","dashboard"].forEach(s=>document.getElementById(s).style.display="none");
  document.getElementById(id).style.display="block";
}

// ---------------- AUTH ----------------
window.register = async function(){
  const email = document.getElementById("registerEmail").value;
  const password = document.getElementById("registerPassword").value;
  try {
    const userCredential = await createUserWithEmailAndPassword(auth,email,password);
    await setDoc(doc(db,"users",userCredential.user.uid),{role:"user"});
    alert("Registered!");
    showSection('dashboard'); loadUserDashboard(userCredential.user.uid);
  } catch(e){
    alert(e.message);
  }
};

window.login = async function(){
  const email = document.getElementById("loginEmail").value;
  const password = document.getElementById("loginPassword").value;
  try {
    const userCredential = await signInWithEmailAndPassword(auth,email,password);
    const user = userCredential.user;
    const docSnap = await getDoc(doc(db,"users",user.uid));
    const role = docSnap.data().role;
    showSection('dashboard');
    role==="admin"? loadAdminDashboard() : loadUserDashboard(user.uid);
  } catch(e){
    alert(e.message);
  }
};

window.logout = function(){
  signOut(auth).then(()=>{
    showSection('login');
    alert("Logged out");
  });
};

// ---------------- ADMIN CRUD w/ Image Upload ----------------
async function loadAdminDashboard(){
  document.getElementById("adminSection").style.display="block";
  document.getElementById("userSection").style.display="none";
  document.getElementById("dashboardTitle").innerText="Admin Dashboard";
  loadProducts();
}

async function loadProducts(){
  const container = document.getElementById("productsContainer");
  container.innerHTML="";
  const querySnapshot = await getDocs(collection(db,"products"));
  querySnapshot.forEach(docSnap=>{
    const p = docSnap.data();
    const id = docSnap.id;
    const div = document.createElement("div");
    div.className = "product-card";
    div.innerHTML = `<img src="${p.image}" alt="${p.name}">
    <b>${p.name}</b><br>KES ${p.price}
    <br><button onclick="deleteProduct('${id}')">Delete</button>
    <button onclick="editProduct('${id}','${p.name}',${p.price},'${p.image}')">Edit</button>`;
    container.appendChild(div);
  });
}

window.addProduct = async function(){
  const name = document.getElementById("productName").value;
  const price = Number(document.getElementById("productPrice").value);
  const fileInput = document.getElementById("productFile");
  if(!name||!price||!fileInput.files[0]) return alert("Fill all fields.");

  // Upload file to Firebase Storage
  const file = fileInput.files[0];
  const storageRef = ref(storage, `products/${Date.now()}_${file.name}`);
  await uploadBytes(storageRef, file);
  const imageUrl = await getDownloadURL(storageRef);

  // Save product
  await addDoc(collection(db,"products"),{name,price,image:imageUrl});
  alert("Product added");
  loadProducts();
};

window.deleteProduct = async function(id){
  await deleteDoc(doc(db,"products",id));
  loadProducts();
};

window.editProduct = async function(id,name,price,image){
  const newName = prompt("Product Name",name);
  const newPrice = prompt("Price (KES)",price);
  const newFile = document.getElementById("productFile").files[0];
  let newUrl = image;

  // If admin selected a new image file, upload it
  if(newFile){
    const storageRef = ref(storage, `products/${Date.now()}_${newFile.name}`);
    await uploadBytes(storageRef,newFile);
    newUrl = await getDownloadURL(storageRef);
  }
  await updateDoc(doc(db,"products",id),{name:newName,price:Number(newPrice),image:newUrl});
  loadProducts();
};

// ---------------- USER DASHBOARD ----------------
let cart = JSON.parse(localStorage.getItem("cart"))||[];

function loadUserDashboard(uid){
  document.getElementById("adminSection").style.display="none";
  document.getElementById("userSection").style.display="block";
  document.getElementById("dashboardTitle").innerText="User Dashboard";
  loadProductsUser(); loadCart();
}

async function loadProductsUser(){
  const container = document.getElementById("productsContainerUser");
  container.innerHTML="";
  const querySnapshot = await getDocs(collection(db,"products"));
  querySnapshot.forEach(docSnap=>{
    const p = docSnap.data();
    const id=docSnap.id;
    const div = document.createElement("div");
    div.className = "product-card";
    div.innerHTML = `<img src="${p.image}" alt="${p.name}">
    <b>${p.name}</b><br>KES ${p.price}<br>
    <button onclick="addToCart('${id}','${p.name}',${p.price},'${p.image}')">Add to Cart</button>`;
    container.appendChild(div);
  });
}

window.addToCart = function(id,name,price,image){
  cart.push({id,name,price,image});
  localStorage.setItem("cart",JSON.stringify(cart));
  loadCart();
};

function loadCart(){
  const container = document.getElementById("cartContainer");
  container.innerHTML="";
  cart.forEach((item,index)=>{
    const div = document.createElement("div");
    div.className = "cart-item";
    div.innerHTML=`<img src="${item.image}" alt="${item.name}">${item.name} - KES ${item.price}
    <button onclick="removeFromCart(${index})">Remove</button>`;
    container.appendChild(div);
  });
}

window.removeFromCart = function(index){
  cart.splice(index,1);
  localStorage.setItem("cart",JSON.stringify(cart));
  loadCart();
};
</script>

</body>
</html>const admin = require('firebase-admin');
const serviceAccount = require('./path-to-service-account.json');

admin.initializeApp({
  credential: admin.credential.cert(serviceAccount)
});

const db = admin.firestore();app.post('/api/stkpush', async (req, res) => {
  const { phone, amount, userId } = req.body;

  // Save order in Firestore
  const orderRef = await db.collection('orders').add({
    userId,
    total: amount,
    status: 'Pending',
    createdAt: admin.firestore.FieldValue.serverTimestamp()
  });

  const accountRef = orderRef.id;

  // Call Safaricom STK Push here (your existing code)
  try {
    const stkResponse = await triggerStkPush(phone, amount, accountRef);
    res.json(stkResponse);
  } catch (err) {
    console.error(err);
    res.status(500).send(err.message);
  }
});
