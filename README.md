<!DOCTYPE html>
<html lang="hi">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>NNCart Pro Admin</title>

<script src="https://cdn.tailwindcss.com"></script>
<script src="https://unpkg.com/html5-qrcode"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>

</head>

<body class="bg-gray-100 p-3">

<h1 class="text-xl font-bold mb-3">🔥 NNCart PRO Admin</h1>

<!-- STATS -->
<div class="grid grid-cols-4 gap-2 mb-3">
<div class="bg-white p-2 text-center shadow rounded"><b id="ordersCount">0</b><br>Orders</div>
<div class="bg-white p-2 text-center shadow rounded"><b id="revenue">0</b><br>Revenue</div>
<div class="bg-white p-2 text-center shadow rounded"><b id="pending">0</b><br>Pending</div>
<div class="bg-white p-2 text-center shadow rounded"><b id="lowStock">0</b><br>Low Stock</div>
</div>

<input id="search" placeholder="Search..." class="border p-2 w-full mb-3 rounded">

<!-- PRODUCT BOX -->
<div class="bg-white p-3 rounded shadow mb-3">

<h2 class="font-bold mb-2">Add / Edit Product</h2>

<input id="barcode" placeholder="Barcode" class="border p-2 w-full mb-1">
<input id="pname" placeholder="Product Name" class="border p-2 w-full mb-1">
<input id="price" placeholder="Price" class="border p-2 w-full mb-1">
<input id="image" placeholder="Image URL" class="border p-2 w-full mb-1">
<input id="category" placeholder="Category" class="border p-2 w-full mb-1">
<input id="stock" placeholder="Stock" class="border p-2 w-full mb-1">
<input id="color" placeholder="Color" class="border p-2 w-full mb-1">
<input id="size" placeholder="Size" class="border p-2 w-full mb-1">

<div class="grid grid-cols-2 gap-2 mt-2">
<button onclick="startScanner()" class="bg-blue-500 text-white p-2 rounded">📷 Scan</button>
<button onclick="saveProduct()" class="bg-green-500 text-white p-2 rounded">💾 Save</button>
<button onclick="clearForm()" class="bg-gray-400 text-white p-2 rounded">Clear</button>
<button onclick="exportData()" class="bg-purple-500 text-white p-2 rounded">Export</button>
</div>

<div id="scanner" class="mt-2"></div>
</div>

<!-- PRODUCTS -->
<h2 class="font-bold">Products</h2>
<div id="products"></div>

<!-- ORDERS -->
<h2 class="font-bold mt-4">Orders</h2>
<div id="orders"></div>

<script type="module">

import { initializeApp } from "https://www.gstatic.com/firebasejs/10.9.0/firebase-app.js";
import { getFirestore, collection, getDocs, addDoc, doc, updateDoc, deleteDoc } from "https://www.gstatic.com/firebasejs/10.9.0/firebase-firestore.js";

const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "nncart.firebaseapp.com",
  projectId: "nncart"
};

const db = getFirestore(initializeApp(firebaseConfig));

let allOrders = [];
let allProducts = [];
let scanner = null;

// ================= LOAD PRODUCTS =================
window.loadProducts = async () => {

  const snap = await getDocs(collection(db,"products"));
  allProducts = snap.docs.map(d => ({id:d.id, ...d.data()}));

  let low = 0;
  let html = "";

  allProducts.forEach(p => {

    if(Number(p.stock) < 5) low++;

    html += `
    <div class="bg-white p-2 mb-2 rounded shadow flex justify-between">

      <div>
        <b>${p["Product Name"]}</b>
        <p class="text-sm">₹${p.Price}</p>
        <p class="text-xs">Stock: ${p.stock}</p>
      </div>

      <div class="flex gap-1">
        <button onclick='editProduct(${JSON.stringify(p)})' class="bg-yellow-400 px-2">Edit</button>
        <button onclick="deleteProduct('${p.id}')" class="bg-red-500 text-white px-2">X</button>
      </div>

    </div>`;
  });

  document.getElementById("products").innerHTML = html;
  document.getElementById("lowStock").innerText = low;
};

// ================= LOAD ORDERS =================
window.loadOrders = async () => {

  const snap = await getDocs(collection(db,"orders"));
  allOrders = snap.docs.map(d => ({id:d.id, ...d.data()}));

  renderOrders(allOrders);
  updateStats();
};

// ================= STATS =================
function updateStats(){

  let rev=0,p=0;

  allOrders.forEach(o=>{
    rev += Number(o.Total||0);
    if(o.Status=="Pending") p++;
  });

  document.getElementById("ordersCount").innerText = allOrders.length;
  document.getElementById("revenue").innerText = "₹"+rev;
  document.getElementById("pending").innerText = p;
}

// ================= ORDERS =================
function renderOrders(data){

  let html="";

  data.forEach(o=>{

    html+=`
    <div class="bg-white p-2 mb-2 rounded shadow">

      <b>${o["Customer Name"]}</b>
      <p>${o.Products}</p>
      <p>₹${o.Total}</p>

      <select onchange="updateStatus('${o.id}',this.value)">
        <option ${o.Status=="Pending"?"selected":""}>Pending</option>
        <option ${o.Status=="Shipped"?"selected":""}>Shipped</option>
        <option ${o.Status=="Delivered"?"selected":""}>Delivered</option>
      </select>

      <input value="${o["Shipping ID"]||""}" 
      onchange="updateShip('${o.id}',this.value)" 
      placeholder="Tracking">

      <div class="flex gap-2 mt-1">
        <button onclick='invoice(${JSON.stringify(o)})' class="bg-purple-500 text-white px-2">PDF</button>
        <button onclick="deleteOrder('${o.id}')" class="bg-red-500 text-white px-2">Delete</button>
      </div>

    </div>`;
  });

  document.getElementById("orders").innerHTML = html;
}

// ================= SEARCH =================
search.oninput = (e)=>{
  let v = e.target.value.toLowerCase();

  renderOrders(allOrders.filter(o =>
    (o["Customer Name"]||"").toLowerCase().includes(v)
  ));
};

// ================= SCANNER =================
window.startScanner = async ()=>{

  if(scanner){
    await scanner.stop();
    scanner.clear();
  }

  scanner = new Html5Qrcode("scanner");

  scanner.start({facingMode:"environment"},{fps:10},

  async(code)=>{

    barcode.value = code;

    try{
      let res = await fetch("https://api.upcitemdb.com/prod/trial/lookup?upc="+code);
      let data = await res.json();

      if(data.items?.length){
        let p = data.items[0];
        pname.value = p.title||"";
        image.value = p.images?.[0]||"";
        price.value = p.offers?.[0]?.price||"";
      }
    }catch{}

    scanner.stop();
  });
};

// ================= SAVE =================
window.saveProduct = async ()=>{

  if(!barcode.value || !pname.value){
    alert("Fill required!");
    return;
  }

  await addDoc(collection(db,"products"),{

    Timestamp: new Date().toISOString(),
    Barcode: barcode.value,
    "Product Name": pname.value,
    "Image URL": image.value,
    Price: Number(price.value||0),
    category: category.value,
    stock: stock.value,
    color: color.value,
    size: size.value

  });

  alert("Saved!");
  clearForm();
  loadProducts();
};

// ================= EDIT =================
window.editProduct = (p)=>{
  barcode.value = p.Barcode;
  pname.value = p["Product Name"];
  price.value = p.Price;
  image.value = p["Image URL"];
  category.value = p.category;
  stock.value = p.stock;
  color.value = p.color;
  size.value = p.size;
};

// ================= DELETE =================
window.deleteProduct = async(id)=>{
  await deleteDoc(doc(db,"products",id));
  loadProducts();
};

window.deleteOrder = async(id)=>{
  await deleteDoc(doc(db,"orders",id));
  loadOrders();
};

// ================= UPDATE =================
window.updateStatus = async(id,s)=>{
  await updateDoc(doc(db,"orders",id),{Status:s});
  loadOrders();
};

window.updateShip = async(id,s)=>{
  await updateDoc(doc(db,"orders",id),{"Shipping ID":s});
};

// ================= CLEAR =================
window.clearForm = ()=>{
  document.querySelectorAll("input").forEach(i=>i.value="");
};

// ================= EXPORT =================
window.exportData = ()=>{
  const data = JSON.stringify(allProducts,null,2);
  const blob = new Blob([data]);
  const a = document.createElement("a");
  a.href = URL.createObjectURL(blob);
  a.download = "products.json";
  a.click();
};

// ================= INVOICE =================
window.invoice = (o)=>{
  const { jsPDF } = window.jspdf;
  const doc = new jsPDF();

  doc.text("NNCart Invoice",20,20);
  doc.text("Name: "+o["Customer Name"],20,40);
  doc.text("Items: "+o.Products,20,50);
  doc.text("Total: ₹"+o.Total,20,60);

  doc.save("invoice.pdf");
};

// ================= START =================
window.onload = ()=>{
  loadProducts();
  loadOrders();
};

setInterval(loadOrders,5000);

</script>

</body>
</html>
