<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Clasique Boutique</title>
<style>
body{font-family:Arial;background:#f4f4f4;margin:0;padding:0;color:#333}
header{background:#000;color:#fff;padding:15px;text-align:center}
header nav a{color:#fff;margin:0 10px;text-decoration:none;font-weight:bold}
header nav a:hover{color:#ffcc00}
footer{background:#000;color:#fff;text-align:center;padding:15px;margin-top:20px}
.container{max-width:1200px;margin:30px auto;padding:10px}
button{background:#000;color:#fff;border:none;padding:10px;border-radius:4px;cursor:pointer}
button:hover{background:#ffcc00;color:#000}
input,select{width:100%;padding:10px;border:1px solid #ccc;border-radius:4px;margin:8px 0}
.products-grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(200px,1fr));gap:20px;margin-top:20px}
.product-card{background:#fff;padding:15px;border-radius:8px;box-shadow:0 4px 8px rgba(0,0,0,0.1);text-align:center}
.cart-item{display:flex;align-items:center;justify-content:space-between;padding:8px;background:#fff;margin-bottom:8px;border-radius:6px;box-shadow:0 2px 5px rgba(0,0,0,0.1)}
.cart-item img{width:50px;height:50px;border-radius:6px;margin-right:10px}
#home,#login,#register,#dashboard{max-width:800px;margin:auto;background:#fff;padding:20px;border-radius:8px;box-shadow:0 4px 8px rgba(0,0,0,0.1)}
</style>
</head>
<body>

<header>
<h1>Clasique Boutique</h1>
<nav>
<a href="#" onclick="showSection('home')">Home</a>
<a href="#" onclick="showSection('login')">Login</a>
<a href="#" onclick="showSection('register')">Register</a>
<a href="#" onclick="showSection('dashboard')">Dashboard</a>
<a href="#" onclick="logout()">Logout</a>
</nav>
<small>📞 0748617221 | 📧 gathunguadams@gmail.com</small>
</header>

<!-- HOME -->
<div id="home" class="container">
  <h2>Welcome to Clasique Boutique</h2>
  <img src="https://i.imgur.com/7qlUwvB.jpg" alt="Boutique Interior" style="width:100%; max-height:400px; object-fit:cover; border-radius:8px; margin-bottom:15px;">
  <p>Browse products, add to cart, and checkout easily. Register only when placing an order.</p>
</div>

<!-- LOGIN -->
<div id="login" class="container" style="display:none">
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

<!-- ADMIN -->
<div id="adminSection" style="display:none">
<h3>Add Product</h3>
<input type="text" id="productName" placeholder="Product Name">
<input type="number" id="productPrice" placeholder="Price (KES)">
<input type="file" id="productFile" accept="image/*">
<button onclick="addProduct()">Add Product</button>

<h3>All Products</h3>
<div id="productsContainer" class="products-grid"></div>
</div>

<!-- USER -->
<div id="userSection" style="display:none">
<h3>Products</h3>
<div id="productsContainerUser" class="products-grid"></div>

<h3>Shopping Cart</h3>
<div id="cartContainer"></div>
</div>

<footer>
© 2026 Clasique Boutique | 0748617221 | gathunguadams@gmail.com
</footer>

<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import { getAuth, createUserWithEmailAndPassword, signInWithEmailAndPassword, signOut } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js";
import { getFirestore, doc, setDoc, getDoc, collection, addDoc, getDocs, updateDoc, deleteDoc } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";
import { getStorage, ref, uploadBytes, getDownloadURL } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-storage.js";

const firebaseConfig = {
  apiKey: "AIzaSyD_QwQycHQuDXEeaEtS19bakyI85DCiFkU",
  authDomain: "clasique-botique.firebaseapp.com",
  projectId: "clasique-botique",
  storageBucket: "clasique-botique.firebasestorage.app",
  messagingSenderId: "810557928750",
  appId: "1:810557928750:web:3d788311afdeba4014b6bf",
  measurementId: "G-SDJ1XMRJ9H"
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const storage = getStorage(app);

function showSection(id){
  ["home","login","register","dashboard"].forEach(s=>document.getElementById(s).style.display="none");
  document.getElementById(id).style.display="block";
}

// REGISTER
window.register = async function(){
  const email = document.getElementById("registerEmail").value;
  const password = document.getElementById("registerPassword").value;
  try {
    const userCredential = await createUserWithEmailAndPassword(auth,email,password);
    await setDoc(doc(db,"users",userCredential.user.uid),{role:"user"});
    alert("Registered!");
    showSection('dashboard'); loadUserDashboard(userCredential.user.uid);
  } catch(e){ alert(e.message); }
};

// LOGIN
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
  } catch(e){ alert(e.message); }
};

// LOGOUT
window.logout = function(){
  signOut(auth).then(()=>{
    showSection('home');
    alert("Logged out");
  });
};

// ADMIN DASHBOARD
async function loadAdminDashboard(){
  document.getElementById("adminSection").style.display="block";
  document.getElementById("userSection").style.display="none";
  document.getElementById("dashboardTitle").innerText="Admin Dashboard";
  loadProductsAdmin();
}

async function loadProductsAdmin(){
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
  const file = fileInput.files[0];
  const storageRef = ref(storage, `products/${Date.now()}_${file.name}`);
  await uploadBytes(storageRef, file);
  const imageUrl = await getDownloadURL(storageRef);
  await addDoc(collection(db,"products"),{name,price,image:imageUrl});
  alert("Product added");
  loadProductsAdmin();
};

window.deleteProduct = async function(id){
  await deleteDoc(doc(db,"products",id));
  loadProductsAdmin();
};

window.editProduct = async function(id,name,price,image){
  const newName = prompt("Product Name",name);
  const newPrice = prompt("Price (KES)",price);
  const newFile = document.getElementById("productFile").files[0];
  let newUrl = image;
  if(newFile){
    const storageRef = ref(storage, `products/${Date.now()}_${newFile.name}`);
    await uploadBytes(storageRef,newFile);
    newUrl = await getDownloadURL(storageRef);
  }
  await updateDoc(doc(db,"products",id),{name:newName,price:Number(newPrice),image:newUrl});
  loadProductsAdmin();
}

// USER DASHBOARD
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
</html>
