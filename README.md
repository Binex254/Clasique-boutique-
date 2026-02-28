<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Clasique Boutique</title>
<style>
body{font-family:Arial;background:#f4f4f4;margin:0}
header{background:#000;color:#fff;padding:15px;text-align:center}
header small{display:block;color:#ffcc00;margin-top:5px}
footer{background:#000;color:#fff;text-align:center;padding:15px;margin-top:30px}
.container{max-width:1100px;margin:auto;padding:20px}
.products{display:grid;grid-template-columns:repeat(auto-fit,minmax(200px,1fr));gap:20px}
.card{background:#fff;padding:15px;border-radius:8px;text-align:center}
button{background:#000;color:#fff;padding:10px;border:none;cursor:pointer}
button:hover{background:#ffcc00;color:#000}
input,select{padding:8px;width:100%;margin:5px 0}
.contact-box{background:#fff;padding:20px;border-radius:8px;margin-top:30px;text-align:center}
</style>
</head>
<body>

<header>
<h2>Clasique Boutique</h2>
<small>📞 0748617221 | 📧 gathunguadams@gmail.com</small>
</header>

<div class="container">

<h3>Products</h3>
<div id="products" class="products"></div>

<h3>Cart</h3>
<div id="cart"></div>

<select id="county">
<option>Nairobi</option>
<option>Kiambu</option>
<option>Mombasa</option>
</select>

<input type="text" id="phone" placeholder="07XXXXXXXX">

<button onclick="checkoutMpesa()">Pay with M-Pesa</button>
<button onclick="checkoutStripe()">Pay with Card (USD)</button>

<div class="contact-box">
<h3>Contact Us</h3>
<p>Phone: 0748617221</p>
<p>Email: <a href="mailto:gathunguadams@gmail.com">gathunguadams@gmail.com</a></p>
</div>

</div>

<footer>
© 2026 Clasique Boutique <br>
📞 0748617221 | 📧 gathunguadams@gmail.com
</footer>

<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import { getFirestore, collection, getDocs } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js";

const firebaseConfig = {
  apiKey: "YOUR_KEY",
  authDomain: "YOUR_AUTH",
  projectId: "YOUR_PROJECT_ID"
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);

let cart=[];
const productsDiv=document.getElementById("products");
const cartDiv=document.getElementById("cart");

async function loadProducts(){
  const snap=await getDocs(collection(db,"products"));
  snap.forEach(doc=>{
    const p=doc.data();
    const div=document.createElement("div");
    div.className="card";
    div.innerHTML=`
      <img src="${p.image}" width="100%">
      <b>${p.name}</b><br>
      KES ${p.price}<br>
      Stock: ${p.stock}<br>
      <button onclick='addToCart("${doc.id}",${p.price})'>Add</button>
    `;
    productsDiv.appendChild(div);
  });
}
loadProducts();

window.addToCart=function(id,price){
  cart.push({id,price});
  renderCart();
}

function renderCart(){
  cartDiv.innerHTML="";
  let total=0;
  cart.forEach(i=>{
    total+=i.price;
    cartDiv.innerHTML+=`Item - KES ${i.price}<br>`;
  });
  cartDiv.innerHTML+=`<b>Subtotal: KES ${total}</b>`;
}

function getDelivery(county){
  const fees={Nairobi:150,Kiambu:200,Mombasa:300};
  return fees[county]||250;
}

window.checkoutMpesa=async function(){
  const county=document.getElementById("county").value;
  const phone=document.getElementById("phone").value;

  const res=await fetch("http://localhost:3000/api/stkpush",{
    method:"POST",
    headers:{"Content-Type":"application/json"},
    body:JSON.stringify({
      userId:"demoUser",
      county,
      phone,
      items:cart
    })
  });

  const data=await res.json();
  alert(data.message);
}

window.checkoutStripe=async function(){
  const res=await fetch("http://localhost:3000/api/stripe-checkout",{
    method:"POST",
    headers:{"Content-Type":"application/json"},
    body:JSON.stringify({amount:50})
  });
  const data=await res.json();
  window.location=data.url;
}
</script>

</body>
</html>const express=require("express");
const admin=require("firebase-admin");
const Stripe=require("stripe");
const PDFDocument=require("pdfkit");
const fs=require("fs");
const cors=require("cors");
const bodyParser=require("body-parser");
const africastalking=require("africastalking");

const app=express();
app.use(cors());
app.use(bodyParser.json());

const stripe=Stripe("YOUR_STRIPE_SECRET");

const at=africastalking({apiKey:"YOUR_AT_KEY",username:"sandbox"});
const sms=at.SMS;

const serviceAccount=require("./serviceAccountKey.json");
admin.initializeApp({credential:admin.credential.cert(serviceAccount)});
const db=admin.firestore();

function deliveryFee(county){const fees={Nairobi:150,Kiambu:200,Mombasa:300};return fees[county]||250;}
async function fraudCheck(userId,total){let score=0;if(total>50000) score+=2;const snap=await db.collection("orders").where("userId","==",userId).get();if(snap.size>5) score+=1;return score;}

app.post("/api/stkpush",async(req,res)=>{
  const {userId,items,county,phone}=req.body;
  const subtotal=items.reduce((s,i)=>s+i.price,0);
  const vat=subtotal*0.16;
  const delivery=deliveryFee(county);
  const total=subtotal+vat+delivery;

  const fraud=await fraudCheck(userId,total);
  if(fraud>=3) return res.status(403).send("Fraud detected");

  const order=await db.collection("orders").add({userId,items,subtotal,vat,delivery,total,status:"Pending"});
  res.json({message:"STK Push Sent",orderId:order.id});
});

app.post("/api/mpesa/callback",async(req,res)=>{
  const {orderId,phone}=req.body;
  const orderDoc=await db.collection("orders").doc(orderId).get();
  const order=orderDoc.data();
  await db.collection("orders").doc(orderId).update({status:"Paid"});
  for(const item of order.items){
    const productRef=db.collection("products").doc(item.id);
    const productDoc=await productRef.get();
    await productRef.update({stock:productDoc.data().stock-1});
  }
  await sms.send({to:phone,message:`Payment received. Order ${orderId} confirmed.`});
  const doc=new PDFDocument();
  doc.pipe(fs.createWriteStream(`receipt_${orderId}.pdf`));
  doc.text("Clasique Boutique Receipt");
  doc.text(`Order ID: ${orderId}`);
  doc.text(`Total: KES ${order.total}`);
  doc.end();
  res.json({ResultCode:0});
});

app.post("/api/stripe-checkout",async(req,res)=>{
  const session=await stripe.checkout.sessions.create({
    payment_method_types:["card"],
    line_items:[{price_data:{currency:"usd",product_data:{name:"Clasique Order"},unit_amount:req.body.amount*100},quantity:1}],
    mode:"payment",
    success_url:"https://yourdomain.com",
    cancel_url:"https://yourdomain.com"
  });
  res.json({url:session.url});
});

app.listen(3000,()=>console.log("Server running on port 3000"));{
  "name": "clasique-boutique",
  "version": "1.0.0",
  "description": "Full Stack eCommerce with M-Pesa and Stripe",
  "main": "server.js",
  "scripts":{"start":"node server.js"},
  "dependencies":{
    "africastalking":"^0.6.6",
    "axios":"^1.6.0",
    "body-parser":"^1.20.2",
    "cors":"^2.8.5",
    "express":"^4.18.2",
    "firebase-admin":"^11.10.1",
    "pdfkit":"^0.13.0",
    "stripe":"^14.0.0"
  }
}# Clasique Boutique

Full Stack eCommerce Platform

Phone: 0748617221  
Email: gathunguadams@gmail.com

## Features
- M-Pesa STK Push
- Stripe International Payments
- VAT Calculation
- Delivery Pricing
- Fraud Detection
- SMS Confirmation
- PDF Receipt
- Stock Management

## Run
cd backend
npm install  
