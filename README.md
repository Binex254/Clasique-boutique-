<HTML
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import { 
  getAuth, 
  createUserWithEmailAndPassword, 
  signInWithEmailAndPassword, 
  signOut,
  onAuthStateChanged
} from "https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js";

import { 
  getFirestore, 
  doc, 
  setDoc, 
  getDoc, 
  collection, 
  addDoc, 
  getDocs, 
  updateDoc, 
  deleteDoc 
} from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";

import { 
  getStorage, 
  ref, 
  uploadBytes, 
  getDownloadURL 
} from "https://www.gstatic.com/firebasejs/10.12.0/firebase-storage.js";

console.log("App Loaded");

// 🔐 Firebase Config
const firebaseConfig = {
  apiKey: "YOUR_NEW_API_KEY",
  authDomain: "clasique-botique.firebaseapp.com",
  projectId: "clasique-botique",
  storageBucket: "clasique-botique.appspot.com",
  messagingSenderId: "810557928750",
  appId: "1:810557928750:web:3d788311afdeba4014b6bf"
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const storage = getStorage(app);

// SECTION SWITCHING
window.showSection = function(id){
  ["home","login","register","dashboard"].forEach(s=>{
    document.getElementById(s).style.display="none";
  });
  document.getElementById(id).style.display="block";
};

// REGISTER
window.register = async function(){
  const email = document.getElementById("registerEmail").value;
  const password = document.getElementById("registerPassword").value;

  if(!email || !password) return alert("Fill all fields");

  try {
    const userCredential = await createUserWithEmailAndPassword(auth,email,password);

    await setDoc(doc(db,"users",userCredential.user.uid),{
      role:"user"
    });

    alert("Registered successfully!");
  } catch(e){
    alert(e.message);
  }
};

// LOGIN
window.login = async function(){
  const email = document.getElementById("loginEmail").value;
  const password = document.getElementById("loginPassword").value;

  if(!email || !password) return alert("Fill all fields");

  try {
    await signInWithEmailAndPassword(auth,email,password);
  } catch(e){
    alert(e.message);
  }
};

// AUTH STATE LISTENER (VERY IMPORTANT)
onAuthStateChanged(auth, async (user) => {
  if(user){
    const docSnap = await getDoc(doc(db,"users",user.uid));

    if(!docSnap.exists()){
      alert("User record missing.");
      return;
    }

    const role = docSnap.data().role;

    showSection("dashboard");

    if(role === "admin"){
      loadAdminDashboard();
    } else {
      loadUserDashboard();
    }

  } else {
    showSection("home");
  }
});

// LOGOUT
window.logout = function(){
  signOut(auth);
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

    div.innerHTML = `
      <img src="${p.image}" width="100%">
      <b>${p.name}</b><br>
      KES ${p.price}<br>
      <button onclick="deleteProduct('${id}')">Delete</button>
    `;

    container.appendChild(div);
  });
}

// ADD PRODUCT
window.addProduct = async function(){
  const name = document.getElementById("productName").value;
  const price = Number(document.getElementById("productPrice").value);
  const file = document.getElementById("productFile").files[0];

  if(!name || !price || !file) return alert("Fill all fields");

  const storageRef = ref(storage, `products/${Date.now()}_${file.name}`);

  await uploadBytes(storageRef,file);
  const imageUrl = await getDownloadURL(storageRef);

  await addDoc(collection(db,"products"),{
    name,
    price,
    image:imageUrl
  });

  alert("Product Added");
  loadProductsAdmin();
};

// DELETE PRODUCT
window.deleteProduct = async function(id){
  await deleteDoc(doc(db,"products",id));
  loadProductsAdmin();
};

// USER DASHBOARD
function loadUserDashboard(){
  document.getElementById("adminSection").style.display="none";
  document.getElementById("userSection").style.display="block";
  document.getElementById("dashboardTitle").innerText="User Dashboard";
  loadProductsUser();
}

async function loadProductsUser(){
  const container = document.getElementById("productsContainerUser");
  container.innerHTML="";

  const querySnapshot = await getDocs(collection(db,"products"));

  querySnapshot.forEach(docSnap=>{
    const p = docSnap.data();

    const div = document.createElement("div");
    div.className="product-card";

    div.innerHTML=`
      <img src="${p.image}" width="100%">
      <b>${p.name}</b><br>
      KES ${p.price}
    `;

    container.appendChild(div);
  });
}
</script>
