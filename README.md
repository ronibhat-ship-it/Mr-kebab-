/*
Mr Kebab - Single-file React App (Tailwind CSS)

Features:
- Full editable menu (categories: Kebabs, Pizzas, Burgers, Tapas, Indian, Paellas, Salads, Pastas, Curries, Drinks)
- CRUD: add, edit, delete menu items with price, category, image (base64)
- 14 table QR code generator (uses Google Chart API to render QR images pointing to menu + table param)
- Waiter order interface: tap items to add to order, send order -> appears in Kitchen Orders list (mocked)
- Photo gallery (upload), social links, map embed, and export/import of menu JSON
- Admin toggle to edit site (no auth—meant as demo; integrate real auth for production)

Dependencies: React, Tailwind CSS. (No external NPM required for QR; uses Google Chart image API.)

How to use:
1. Create a React app (Vite or CRA) with Tailwind installed.
2. Copy this file into src/components/MrKebabApp.jsx and import it in App.jsx.
3. Run with `npm run dev` or `npm start`.

Notes:
- For real kitchen integration you should hook "sendOrderToKitchen" to your POS/kitchen printer API or a backend.
- Replace Google Maps iframe src with your exact location or use the Maps JavaScript API.
*/

import React, { useState, useEffect } from "react";

export default function MrKebabApp() {
  // sample menu data
  const defaultMenu = [
    { id: 1, category: "Kebabs", name: "Pita (Chicken)", price: 5.5, notes: "Chicken, fries option", image: null },
    { id: 2, category: "Kebabs", name: "Durum (Mixed)", price: 6.5, notes: "Chicken & beef mixed", image: null },
    { id: 3, category: "Pizzas", name: "Margarita", price: 8.95, notes: "Tomato, cheese", image: null },
    { id: 4, category: "Burgers", name: "Chicken burger", price: 9.95, notes: "With fries", image: null },
    { id: 5, category: "Tapas", name: "Onion rings", price: 6.95, notes: "", image: null },
    { id: 6, category: "Indian", name: "Chicken curry", price: 11.95, notes: "With rice", image: null },
    { id: 7, category: "Paellas", name: "Seafood paella", price: 13.5, notes: "Saffron", image: null },
    { id: 8, category: "Salads", name: "Caesar salad", price: 9.5, notes: "", image: null },
    { id: 9, category: "Pastas", name: "Spaghetti bolognese", price: 9.5, notes: "", image: null },
    { id: 10, category: "Curries", name: "Biryani de pollo", price: 11.95, notes: "Chicken biryani", image: null },
    { id: 11, category: "Drinks", name: "Coca-Cola 330ml", price: 2.5, notes: "", image: null },
    { id: 12, category: "Kebabs", name: "Falafel Durum", price: 6.0, notes: "Vegetarian option", image: null },
  ];

  const [menu, setMenu] = useState(() => {
    try {
      const saved = localStorage.getItem("mrkebab_menu");
      return saved ? JSON.parse(saved) : defaultMenu;
    } catch (e) {
      return defaultMenu;
    }
  });

  const [gallery, setGallery] = useState(() => {
    const saved = localStorage.getItem("mrkebab_gallery");
    return saved ? JSON.parse(saved) : [];
  });

  const categories = [
    "Kebabs",
    "Pizzas",
    "Burgers",
    "Tapas",
    "Indian",
    "Paellas",
    "Salads",
    "Pastas",
    "Curries",
    "Drinks",
  ];

  useEffect(() => {
    localStorage.setItem("mrkebab_menu", JSON.stringify(menu));
  }, [menu]);

  useEffect(() => {
    localStorage.setItem("mrkebab_gallery", JSON.stringify(gallery));
  }, [gallery]);

  // Admin form state
  const [adminMode, setAdminMode] = useState(true); // toggle to show admin controls
  const [editing, setEditing] = useState(null);
  const [form, setForm] = useState({ category: "Kebabs", name: "", price: "", notes: "", image: null });

  // Waiter / ordering
  const [currentTable, setCurrentTable] = useState(1);
  const [order, setOrder] = useState([]);
  const [kitchenOrders, setKitchenOrders] = useState(() => {
    const saved = localStorage.getItem("mrkebab_kitchen");
    return saved ? JSON.parse(saved) : [];
  });

  useEffect(() => {
    localStorage.setItem("mrkebab_kitchen", JSON.stringify(kitchenOrders));
  }, [kitchenOrders]);

  // utility ID generator
  const nextId = () => Math.max(0, ...menu.map((m) => m.id)) + 1;

  function handleImageUpload(e, cb) {
    const file = e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => cb(reader.result);
    reader.readAsDataURL(file);
  }

  function addMenuItem() {
    if (!form.name || !form.price) return alert("Name and price required");
    const item = { id: nextId(), category: form.category, name: form.name, price: parseFloat(form.price), notes: form.notes, image: form.image };
    setMenu((m) => [...m, item]);
    setForm({ category: "Kebabs", name: "", price: "", notes: "", image: null });
  }

  function startEdit(item) {
    setEditing(item.id);
    setForm({ category: item.category, name: item.name, price: item.price, notes: item.notes, image: item.image });
  }

  function applyEdit() {
    setMenu((m) => m.map((it) => (it.id === editing ? { ...it, category: form.category, name: form.name, price: parseFloat(form.price), notes: form.notes, image: form.image } : it)));
    setEditing(null);
    setForm({ category: "Kebabs", name: "", price: "", notes: "", image: null });
  }

  function deleteItem(id) {
    if (!confirm("Delete this item?")) return;
    setMenu((m) => m.filter((it) => it.id !== id));
  }

  function uploadGalleryImage(e) {
    handleImageUpload(e, (data) => {
      setGallery((g) => [...g, { id: Date.now(), src: data }]);
    });
  }

  function addToOrder(item) {
    setOrder((o) => [...o, { ...item, qty: 1 }]);
  }

  function changeQty(index, delta) {
    setOrder((o) => o.map((it, i) => (i === index ? { ...it, qty: Math.max(1, it.qty + delta) } : it)));
  }

  function removeFromOrder(index) {
    setOrder((o) => o.filter((_, i) => i !== index));
  }

  function sendOrderToKitchen() {
    if (order.length === 0) return alert("No items in order");
    const payload = { id: Date.now(), table: currentTable, items: order, time: new Date().toISOString(), status: "pending" };
    setKitchenOrders((k) => [payload, ...k]);
    setOrder([]);
  }

  function markKitchenDone(id) {
    setKitchenOrders((k) => k.map((o) => (o.id === id ? { ...o, status: "done" } : o)));
  }

  function exportMenu() {
    const blob = new Blob([JSON.stringify({ menu, gallery }, null, 2)], { type: "application/json" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "mrkebab_menu_export.json";
    a.click();
  }

  function importMenu(e) {
    const file = e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => {
      try {
        const data = JSON.parse(reader.result);
        if (data.menu) setMenu(data.menu);
        if (data.gallery) setGallery(data.gallery);
        alert("Imported");
      } catch (err) {
        alert("Invalid file");
      }
    };
    reader.readAsText(file);
  }

  // Generate QR image URL for a table
  function qrForTable(table) {
    const siteUrl = window.location.href.split("?")[0];
    const url = `${siteUrl}?table=${table}`;
    return `https://chart.googleapis.com/chart?cht=qr&chs=300x300&chl=${encodeURIComponent(url)}`;
  }

  // Filter menu by search & category
  const [search, setSearch] = useState("");
  const [filterCategory, setFilterCategory] = useState("");

  const filteredMenu = menu.filter((mItem) => {
    if (filterCategory && mItem.category !== filterCategory) return false;
    if (!search) return true;
    const s = search.toLowerCase();
    return mItem.name.toLowerCase().includes(s) || (mItem.notes || "").toLowerCase().includes(s);
  });

  return (
    <div className="min-h-screen bg-gradient-to-b from-orange-50 to-white p-6">
      <div className="max-w-6xl mx-auto">
        <header className="flex items-center justify-between mb-6">
          <div>
            <h1 className="text-4xl font-extrabold">Mr Kebab <span className="text-lg font-medium ml-2">— Kebab & Pizzeria</span></h1>
            <p className="text-sm text-gray-600">Delicious kebabs, pizzas, tapas & more — edit your menu, generate table QR codes, and take orders.</p>
          </div>
          <div className="flex items-center gap-3">
            <button onClick={() => setAdminMode((s) => !s)} className="px-3 py-1 rounded bg-gray-800 text-white text-sm">{adminMode ? "Admin: ON" : "Admin: OFF"}</button>
            <button onClick={exportMenu} className="px-3 py-1 rounded border">Export</button>
            <label className="px-3 py-1 rounded border cursor-pointer">
              Import
              <input type="file" accept="application/json" onChange={importMenu} className="hidden" />
            </label>
          </div>
        </header>

        <main className="grid grid-cols-1 md:grid-cols-3 gap-6">
          {/* Left: Menu & ordering */}
          <section className="col-span-2 bg-white p-4 rounded shadow">
            <div className="flex items-center gap-3 mb-4">
              <input value={search} onChange={(e) => setSearch(e.target.value)} placeholder="Search menu..." className="flex-1 p-2 border rounded" />
              <select value={filterCategory} onChange={(e) => setFilterCategory(e.target.value)} className="p-2 border rounded">
                <option value="">All categories</option>
                {categories.map((c) => <option key={c} value={c}>{c}</option>)}
              </select>
            </div>

            <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
              {filteredMenu.map((item) => (
                <div key={item.id} className="border rounded p-3 flex gap-3 items-center">
                  <div className="w-20 h-20 bg-gray-100 flex items-center justify-center rounded overflow-hidden">
                    {item.image ? <img src={item.image} alt="item" className="w-full h-full object-cover" /> : <span className="text-xs text-gray-400">No image</span>}
                  </div>
                  <div className="flex-1">
                    <div className="flex items-center justify-between">
                      <div>
                        <div className="font-semibold">{item.name}</div>
                        <div className="text-xs text-gray-500">{item.category} · {item.notes}</div>
                      </div>
                      <div className="text-right">
                        <div className="font-bold">€{Number(item.price).toFixed(2)}</div>
                        <div className="text-xs text-gray-500">ID:{item.id}</div>
                      </div>
                    </div>

                    <div className="mt-2 flex gap-2">
                      <button onClick={() => addToOrder(item)} className="px-3 py-1 rounded bg-green-600 text-white text-sm">Add to order</button>
                      {adminMode && (
                        <>
                          <button onClick={() => startEdit(item)} className="px-2 py-1 rounded border text-sm">Edit</button>
                          <button onClick={() => deleteItem(item.id)} className="px-2 py-1 rounded border text-sm text-red-600">Delete</button>
                        </>
                      )}
                    </div>
                  </div>
                </div>
              ))}
            </div>

            {/* Order panel */}
            <div className="mt-6 bg-gray-50 p-3 rounded">
              <h3 className="font-semibold">Waiter Order</h3>
              <div className="flex items-center gap-2 mt-2">
                <label className="text-sm">Table:</label>
                <select value={currentTable} onChange={(e) => setCurrentTable(Number(e.target.value))} className="p-1 border rounded">
                  {Array.from({ length: 14 }).map((_, i) => <option key={i+1} value={i+1}>{i+1}</option>)}
                </select>
                <button onClick={() => { navigator.clipboard?.writeText(window.location.href + `?table=${currentTable}`); alert('Menu link copied to clipboard'); }} className="ml-2 px-2 py-1 border rounded text-xs">Copy menu link</button>
                <div className="ml-auto text-sm text-gray-600">Items: {order.length}</div>
              </div>

              <div className="mt-3">
                {order.length === 0 ? <div className="text-sm text-gray-500">No items added</div> : (
                  <div>
                    {order.map((it, idx) => (
                      <div key={idx} className="flex items-center justify-between p-2 border-b">
                        <div>
                          <div className="font-medium">{it.name}</div>
                          <div className="text-xs text-gray-500">€{it.price.toFixed(2)}</div>
                        </div>
                        <div className="flex items-center gap-2">
                          <button onClick={() => changeQty(idx, -1)} className="px-2 py-1 border rounded text-sm">-</button>
                          <div>{it.qty}</div>
                          <button onClick={() => changeQty(idx, 1)} className="px-2 py-1 border rounded text-sm">+</button>
                          <button onClick={() => removeFromOrder(idx)} className="px-2 py-1 border rounded text-sm text-red-600">Remove</button>
                        </div>
                      </div>
                    ))}

                    <div className="mt-3 flex items-center justify-between">
                      <div className="text-sm font-semibold">Total: €{order.reduce((s, it) => s + it.price * it.qty, 0).toFixed(2)}</div>
                      <div className="flex gap-2">
                        <button onClick={sendOrderToKitchen} className="px-3 py-1 rounded bg-blue-600 text-white">Send to Kitchen</button>
                        <button onClick={() => setOrder([])} className="px-3 py-1 rounded border">Clear</button>
                      </div>
                    </div>
                  </div>
                )}
              </div>
            </div>

          </section>

          {/* Right: Admin / Gallery / QR / Kitchen */}
          <aside className="bg-white p-4 rounded shadow">
            <div className="mb-4">
              <h3 className="font-semibold">Kitchen Orders</h3>
              <div className="mt-2 max-h-40 overflow-auto">
                {kitchenOrders.length === 0 ? <div className="text-sm text-gray-500">No orders</div> : kitchenOrders.map((o) => (
                  <div key={o.id} className={`p-2 border rounded mb-2 ${o.status === 'done' ? 'bg-green-50' : ''}`}>
                    <div className="flex items-center justify-between">
                      <div className="text-sm">Table {o.table} · {new Date(o.time).toLocaleTimeString()}</div>
                      <div className="text-xs text-gray-500">{o.status}</div>
                    </div>
                    <div className="mt-2 text-sm">
                      {o.items.map((it, i) => <div key={i}>{it.qty}x {it.name}</div>)}
                    </div>
                    <div className="mt-2 flex gap-2">
                      <button onClick={() => markKitchenDone(o.id)} className="px-2 py-1 border rounded text-xs">Mark done</button>
                    </div>
                  </div>
                ))}
              </div>
            </div>

            <div className="mb-4">
              <h3 className="font-semibold">Table QR Codes</h3>
              <div className="grid grid-cols-2 gap-2 mt-2">
                {Array.from({ length: 14 }).map((_, i) => (
                  <div key={i} className="p-2 border rounded text-center">
                    <div className="text-sm">Table {i+1}</div>
                    <img src={qrForTable(i+1)} alt={`QR ${i+1}`} className="mx-auto w-28 h-28" />
                    <div className="mt-1 text-xs text-gray-500">Right-click save to print</div>
                  </div>
                ))}
              </div>
            </div>

            <div className="mb-4">
              <h3 className="font-semibold">Gallery</h3>
              <input type="file" accept="image/*" onChange={uploadGalleryImage} className="mt-2" />
              <div className="mt-2 grid grid-cols-3 gap-2">
                {gallery.map((g) => <img key={g.id} src={g.src} alt="gallery" className="w-full h-16 object-cover rounded" />)}
              </div>
            </div>

            <div>
              <h3 className="font-semibold">Location & Social</h3>
              <div className="mt-2 text-sm text-gray-600">C/ BARBASTRE N°9 LOCAL 10 SALOU · C/ JOSEP CARNER 17 LOCAL 4 SALOU</div>
              <div className="mt-2">
                <iframe title="map" src="https://www.google.com/maps/embed?pb=!1m18!1m12!1m3!1d..." className="w-full h-32 border rounded" />
                <div className="flex gap-2 mt-2">
                  <a href="#" className="text-sm underline">Instagram</a>
                  <a href="#" className="text-sm underline">Facebook</a>
                  <a href="#" className="text-sm underline">WhatsApp</a>
                </div>
              </div>
            </div>

          </aside>
        </main>

        {/* Admin editor modal area */}
        {adminMode && (
          <div className="mt-6 bg-white p-4 rounded shadow">
            <h3 className="font-semibold">Admin: Add / Edit Menu Items</h3>
            <div className="grid grid-cols-1 md:grid-cols-3 gap-3 mt-3">
              <select className="p-2 border rounded" value={form.category} onChange={(e) => setForm((f) => ({ ...f, category: e.target.value }))}>
                {categories.map((c) => <option key={c} value={c}>{c}</option>)}
              </select>
              <input placeholder="Item name" value={form.name} onChange={(e) => setForm((f) => ({ ...f, name: e.target.value }))} className="p-2 border rounded" />
              <input placeholder="Price" value={form.price} onChange={(e) => setForm((f) => ({ ...f, price: e.target.value }))} className="p-2 border rounded" />
              <input placeholder="Notes" value={form.notes} onChange={(e) => setForm((f) => ({ ...f, notes: e.target.value }))} className="p-2 border rounded md:col-span-2" />
              <div className="md:col-span-1">
                <label className="block p-2 border rounded cursor-pointer text-sm">Upload image (optional)
                  <input type="file" accept="image/*" onChange={(e) => handleImageUpload(e, (data) => setForm((f) => ({ ...f, image: data })))} className="hidden" />
                </label>
                {form.image && <img src={form.image} alt="preview" className="w-24 h-24 object-cover mt-2 rounded" />}
              </div>
            </div>
            <div className="mt-3 flex gap-2">
              {editing ? (
                <>
                  <button onClick={applyEdit} className="px-3 py-1 rounded bg-yellow-500">Apply edit</button>
                  <button onClick={() => { setEditing(null); setForm({ category: "Kebabs", name: "", price: "", notes: "", image: null }); }} className="px-3 py-1 rounded border">Cancel</button>
                </>
              ) : (
                <button onClick={addMenuItem} className="px-3 py-1 rounded bg-green-600 text-white">Add item</button>
              )}
            </div>
          </div>
        )}

        {/* Footer */}
        <footer className="mt-8 text-center text-gray-600 text-sm">
          © Mr Kebab — Editable demo site. Replace this with live domain and secure admin authentication for production.
        </footer>

      </div>
    </div>
  );
}


