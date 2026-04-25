<!DOCTYPE html>
<html lang="hi">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">

<title>NNCart Ultimate Admin</title>

<script src="https://cdn.tailwindcss.com"></script>
<script src="https://unpkg.com/html5-qrcode"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>

<style>
body{background:#f3f4f6}
</style>

</head>

<body class="p-3">

<h1 class="text-xl font-bold mb-3">NNCart Admin Panel</h1>

<!-- STATS -->
<div class="grid grid-cols-3 gap-2 mb-3">
<div class="bg-white p-2 text-center shadow rounded"><b id="ordersCount">0</b><br>Orders</div>
<div class="bg-white p-2 text-center shadow rounded"><b id="revenue">0</b><br>Revenue</div>
<div class="bg-white p-2 text-center shadow rounded"><b id="pending">0</b><br>Pending</div>
</div>

<input id="search" placeholder="Search Customer..." class="border p-2 w-full mb-3 rounded">

<!-- PRODUCT ADD -->
<div class="bg-white p-3 rounded shadow mb-3">
<h2 class="font-bold mb-2">Add / Scan Product</h2>

<input id="barcode" placeholder="Barcode" class="border p-2 w-full mb-1 rounded">
<input id="pname" placeholder="Product Name" class="border p-2 w-full mb-1 rounded">
<input id="price" placeholder="Price" class="border p-2 w-full mb-1 rounded">
<input id="image" placeholder="Image URL" class="border p-2 w-full mb-1 rounded">
<input id="category" placeholder="Category" class="border p-2 w-full mb-1 rounded">
<input id="stock" placeholder="Stock" class="border p-2 w-full mb-1 rounded">
<input id="color" placeholder="Color" class="border p-2 w-full mb-1 rounded">
<input id="size" placeholder="Size" class="border p-2 w-full mb-1 rounded">

<div class="flex gap-2 mt-2">
<button onclick="startScanner()" class="bg-blue-500 text-white px-3 py-2 rounded w-full">📷 Scan</button>
<button onclick="saveProduct()" class="bg-green-500 text-white px-3 py-2 rounded w-full">💾 Save</button>
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
let scannerInstance = null;

// ================= LOAD PRODUCTS =================
window.loadProducts = async () => {
  const snap = await getDocs(collection(db, "products"));

  let html = "";

  snap.forEach(d => {
    let p = d.data();

    html += `
    <div class="bg-white p-2 mb-2 rounded shadow flex justify-between items-center">

      <div>
        <p class="font-bold">${p["Product Name"] || "No Name"}</p>
        <p class="text-sm text-gray-500">₹${p["Price"] || 0}</p>
      </div>

      <button onclick="deleteProduct('${d.id}')"
      class="bg-red-500 text-white px-2 py-1 rounded text-sm">Delete</button>

    </div>`;
  });

  document.getElementById("products").innerHTML = html;
};

// ================= LOAD ORDERS =================
window.loadOrders = async () => {

  const snap = await getDocs(collection(db, "orders"));

  allOrders = snap.docs.map(d => ({ id: d.id, ...d.data() }));

  renderOrders(allOrders);
  updateStats(allOrders);
};

// ================= RENDER ORDERS =================
function renderOrders(data){

  let html = "";

  data.forEach(o => {

    html += `
    <div class="bg-white p-2 mb-2 rounded shadow">

      <b>${o["Customer Name"] || ""}</b>
      <p class="text-sm">${o["Products"] || ""}</p>

      <p class="font-bold">₹${o["Total"] || 0}</p>

      <p class="text-xs">Status: ${o["Status"]}</p>

      <select onchange="updateStatus('${o.id}', this.value)"
      class="border p-1 mt-1">

        <option ${o["Status"]=="Pending"?"selected":""}>Pending</option>
        <option ${o["Status"]=="Shipped"?"selected":""}>Shipped</option>
        <option ${o["Status"]=="Delivered"?"selected":""}>Delivered</option>

      </select>

      <input value="${o["Shipping ID"] || ""}"
      onchange="updateShip('${o.id}', this.value)"
      placeholder="Shipping ID"
      class="border p-1 mt-1 w-full">

      <div class="flex gap-2 mt-2">

        ${o["Payment Proof"] ? `<a href="${o["Payment Proof"]}" target="_blank" class="text-blue-500 text-xs">Payment</a>` : ""}

        <button onclick='invoice(${JSON.stringify(o)})'
        class="bg-purple-500 text-white px-2 py-1 text-xs rounded">Invoice</button>

        <button onclick="deleteOrder('${o.id}')"
        class="bg-red-500 text-white px-2 py-1 text-xs rounded">Delete</button>

      </div>

    </div>`;
  });

  document.getElementById("orders").innerHTML = html;
}

// ================= STATS =================
function updateStats(d){

  document.getElementById("ordersCount").innerText = d.length;

  let rev = 0, p = 0;

  d.forEach(o=>{
    rev += Number(o["Total"]||0);
    if(o["Status"]=="Pending") p++;
  });

  document.getElementById("revenue").innerText = "₹"+rev;
  document.getElementById("pending").innerText = p;
}

// ================= SEARCH =================
document.getElementById("search").oninput = (e)=>{
  let v = e.target.value.toLowerCase();

  renderOrders(
    allOrders.filter(o => (o["Customer Name"]||"").toLowerCase().includes(v))
  );
};

// ================= SCANNER =================
window.startScanner = async ()=>{

  if(scannerInstance){
    await scannerInstance.stop();
    scannerInstance.clear();
  }

  scannerInstance = new Html5Qrcode("scanner");

  scannerInstance.start(
    { facingMode:"environment" },
    { fps:10 },

    async(code)=>{

      document.getElementById("barcode").value = code;

      // 🔥 API FETCH
      try{
        const res = await fetch("https://api.upcitemdb.com/prod/trial/lookup?upc="+code);
        const data = await res.json();

        if(data.items && data.items.length){

          const p = data.items[0];

          document.getElementById("pname").value = p.title || "";
          document.getElementById("image").value = p.images?.[0] || "";
          document.getElementById("price").value = p.offers?.[0]?.price || "";

        }

      }catch(e){
        console.log("API fail");
      }

      await scannerInstance.stop();
    }
  );
};

// ================= SAVE PRODUCT =================
window.saveProduct = async ()=>{

  if(!barcode.value || !pname.value){
    alert("Barcode & Name required!");
    return;
  }

  await addDoc(collection(db,"products"),{

    "Timestamp": new Date().toISOString(),
    "Barcode": barcode.value,
    "Product Name": pname.value,
    "Image URL": image.value,
    "Price": Number(price.value||0),
    "category": category.value,
    "stock": stock.value,
    "color": color.value,
    "size": size.value

  });

  alert("Saved ✅");

  document.querySelectorAll("input").forEach(i=>i.value="");

  loadProducts();
};

// ================= UPDATE =================
window.updateStatus = async(id,s)=>{
  await updateDoc(doc(db,"orders",id),{"Status":s});
  loadOrders();
};

window.updateShip = async(id,s)=>{
  await updateDoc(doc(db,"orders",id),{"Shipping ID":s});
};

window.deleteOrder = async(id)=>{
  await deleteDoc(doc(db,"orders",id));
  loadOrders();
};

window.deleteProduct = async(id)=>{
  await deleteDoc(doc(db,"products",id));
  loadProducts();
};

// ================= INVOICE =================
window.invoice = (o)=>{
  const { jsPDF } = window.jspdf;

  const doc = new jsPDF();

  doc.text("NNCart Invoice",20,20);
  doc.text("Customer: "+o["Customer Name"],20,40);
  doc.text("Product: "+o["Products"],20,50);
  doc.text("Total: ₹"+o["Total"],20,60);

  doc.save("invoice.pdf");
};

// ================= START =================
window.onload = ()=>{
  loadOrders();
  loadProducts();
};

setInterval(loadOrders,5000);

</script>

</body>
</html>
