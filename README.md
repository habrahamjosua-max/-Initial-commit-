import React, { useEffect, useState } from "react";

// Rifas Toluca - Single-file React component (Tailwind CSS assumed available)
// Funcionalidades incluidas (simuladas en frontend / localStorage):
// - 100000 boletos numerados del 1 al 100000
// - Registro de comprador con número de WhatsApp
// - Compra (selección de cantidad) y asignación automática de números de boleto
// - Generación de link para notificación por WhatsApp (abre WhatsApp con el mensaje prellenado)
// - Vista de administrador en /admin-rifas-toluca protegida por contraseña (contraseña por defecto: JosuaJa22$)
// - Cálculo automático del premio: 80% de la recaudación (editable)
// - Sorteo aleatorio entre boletos vendidos
// Nota: Esto es una versión frontend-demo. Para producción se necesita backend + base de datos + pasarela de pagos + servicio real de envío de WhatsApp.

const TOTAL_TICKETS = 100000;
const ADMIN_PATH = "/admin-rifas-toluca";
const ADMIN_PASSWORD_DEFAULT = "JosuaJa22$";
const STORAGE_KEY = "rifas_toluca_v1";

function makeInitialState() {
  // estado guardado en localStorage: {ticketsSold: Set, purchases: [], settings:{price, prizePct}}
  const purchases = [];
  const ticketsSold = {};
  return { ticketsSold, purchases, settings: { price: 20, prizePct: 80 } };
}

function loadState() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return makeInitialState();
    const parsed = JSON.parse(raw);
    // convert ticketsSold list to map for quick lookup
    const ts = {};
    (parsed.ticketsSold || []).forEach((t) => (ts[t] = true));
    return { ticketsSold: ts, purchases: parsed.purchases || [], settings: parsed.settings || { price: 20, prizePct: 80 } };
  } catch (e) {
    console.error(e);
    return makeInitialState();
  }
}

function saveState(state) {
  const ticketsSoldList = Object.keys(state.ticketsSold).map((k) => Number(k));
  const toSave = { ticketsSold: ticketsSoldList, purchases: state.purchases, settings: state.settings };
  localStorage.setItem(STORAGE_KEY, JSON.stringify(toSave));
}

export default function RifasToluca() {
  const [state, setState] = useState(() => loadState());
  const [route, setRoute] = useState(window.location.pathname);
  const [buyerWhatsApp, setBuyerWhatsApp] = useState("");
  const [qty, setQty] = useState(1);
  const [buyerName, setBuyerName] = useState("");
  const [message, setMessage] = useState("");
  const [adminPassword, setAdminPassword] = useState("");
  const [filter, setFilter] = useState("");

  useEffect(() => {
    const onPop = () => setRoute(window.location.pathname);
    window.addEventListener("popstate", onPop);
    return () => window.removeEventListener("popstate", onPop);
  }, []);

  useEffect(() => saveState(state), [state]);

  const soldCount = Object.keys(state.ticketsSold).length;
  const remaining = TOTAL_TICKETS - soldCount;

  function pushRoute(p) {
    window.history.pushState({}, "", p);
    setRoute(p);
  }

  function formatCurrency(n) {
    return Number(n).toLocaleString("es-MX", { style: "currency", currency: "MXN" });
  }

  function buyTickets() {
    if (!buyerWhatsApp) {
      setMessage("Ingrese su número de WhatsApp (solo dígitos, con clave de país, ej: 521341...) ");
      return;
    }
    if (qty < 1 || qty > 1000) {
      setMessage("Cantidad inválida (mín 1, máx 1000 por transacción en demo)." );
      return;
    }
    if (qty > remaining) {
      setMessage("No quedan suficientes boletos disponibles.");
      return;
    }

    // asignar boletos aleatorios disponibles
    const assigned = [];
    // simple strategy: scan from random start until find enough
    let attempts = 0;
    while (assigned.length < qty && attempts < TOTAL_TICKETS * 2) {
      const candidate = Math.floor(Math.random() * TOTAL_TICKETS) + 1;
      if (!state.ticketsSold[candidate]) {
        state.ticketsSold[candidate] = true;
        assigned.push(candidate);
      }
      attempts++;
    }
    // si algo falló (posible en rare cases), completar con búsqueda secuencial
    if (assigned.length < qty) {
      for (let i = 1; i <= TOTAL_TICKETS && assigned.length < qty; i++) {
        if (!state.ticketsSold[i]) {
          state.ticketsSold[i] = true;
          assigned.push(i);
        }
      }
    }

    const totalPrice = qty * state.settings.price;
    const purchase = {
      id: Date.now() + "_" + Math.floor(Math.random()*1000),
      name: buyerName || null,
      whatsapp: buyerWhatsApp,
      tickets: assigned.sort((a,b)=>a-b),
      qty,
      totalPrice,
      paid: false,
      createdAt: new Date().toISOString(),
    };

    const newState = { ...state, purchases: [...state.purchases, purchase], ticketsSold: { ...state.ticketsSold } };
    setState(newState);
    setMessage("");

    // preparar mensaje de WhatsApp con los números de boleto
    const ticketsText = purchase.tickets.join(", ");
    const text = `¡Gracias por tu compra!\nBoletos: ${ticketsText}\nTotal: ${formatCurrency(totalPrice)}\nRifas Toluca — ¡Buena suerte!`;
    const waUrl = `https://wa.me/${buyerWhatsApp.replace(/[^0-9]/g,"")}?text=${encodeURIComponent(text)}`;

    // abrir WhatsApp Web/APP en nueva pestaña para que el usuario envíe el mensaje (simula notificación)
    window.open(waUrl, "_blank");

    // mostrar confirmación en la UI
    setMessage(`Boletos asignados: ${ticketsText}. Se abrió WhatsApp para notificación. Registra el pago para confirmar.`);
    setBuyerWhatsApp("");
    setBuyerName("");
    setQty(1);
  }

  function adminLogin() {
    if (adminPassword === ADMIN_PASSWORD_DEFAULT) {
      pushRoute(ADMIN_PATH + "#logged");
      setAdminPassword("");
    } else {
      alert("Contraseña incorrecta");
    }
  }

  function markPaid(purchaseId) {
    const purchases = state.purchases.map((p) => (p.id === purchaseId ? { ...p, paid: true } : p));
    setState({ ...state, purchases });
  }

  function refundPurchase(purchaseId) {
    // demo: marcar como no pagado y liberar boletos (nota: en reglas reales los pagos no son reembolsables)
    if (!confirm("Esta acción liberará los boletos al pool y marcará la compra como no pagada. ¿Continuar?")) return;
    const purchases = [];
    const ticketsSold = { ...state.ticketsSold };
    state.purchases.forEach((p) => {
      if (p.id === purchaseId) {
        p.tickets.forEach((t) => delete ticketsSold[t]);
        // omit this purchase (or mark refunded)
      } else {
        purchases.push(p);
      }
    });
    setState({ ...state, purchases, ticketsSold });
  }

  function drawWinner() {
    // elegir ganador aleatorio entre todos los boletos vendidos
    const allSold = [];
    state.purchases.forEach((p) => p.tickets.forEach((t) => allSold.push({ ticket: t, buyer: p }))); 
    if (allSold.length === 0) { alert("No hay boletos vendidos."); return; }
    const winner = allSold[Math.floor(Math.random() * allSold.length)];
    const totalCollected = state.purchases.reduce((s,p) => s + (p.qty * state.settings.price), 0);
    const prize = Math.round((state.settings.prizePct/100) * totalCollected);
    alert(`¡Ganador! Boleto #${winner.ticket}\nComprador: ${winner.buyer.name || winner.buyer.whatsapp}\nPremio: ${formatCurrency(prize)}`);
  }

  function exportSalesCSV() {
    const headers = ["id","name","whatsapp","tickets","qty","totalPrice","paid","createdAt"];
    const rows = state.purchases.map(p => [p.id, p.name||"", p.whatsapp, `"${p.tickets.join("|")}"`, p.qty, p.totalPrice, p.paid, p.createdAt]);
    const csv = [headers.join(","), ...rows.map(r=>r.join(","))].join("\n");
    const blob = new Blob([csv], { type: 'text/csv' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'rifas_toluca_ventas.csv';
    a.click();
    URL.revokeObjectURL(url);
  }

  // Render
  return (
    <div className="min-h-screen bg-slate-50 p-6">
      <div className="max-w-4xl mx-auto">
        <header className="flex items-center justify-between mb-6">
          <div>
            <h1 className="text-3xl font-bold">Rifas Toluca — Pulsar RS200</h1>
            <p className="text-sm text-gray-600">Boleto total disponible: {TOTAL_TICKETS.toLocaleString()} — Quedan: {remaining.toLocaleString()}</p>
          </div>
          <div className="text-right">
            <button className="px-3 py-1 rounded bg-indigo-600 text-white" onClick={()=>pushRoute('/')}>Inicio</button>
            <button className="ml-2 px-3 py-1 rounded border" onClick={()=>pushRoute(ADMIN_PATH)}>Admin</button>
          </div>
        </header>

        {route === '/' && (
          <main className="grid grid-cols-1 md:grid-cols-2 gap-6">
            <section className="bg-white p-4 rounded shadow">
              <h2 className="text-xl font-semibold mb-2">Comprar boletos</h2>
              <label className="block text-sm">Nombre (opcional)</label>
              <input className="w-full p-2 border rounded mb-2" value={buyerName} onChange={e=>setBuyerName(e.target.value)} />

              <label className="block text-sm">WhatsApp (ej: 521341...)</label>
              <input className="w-full p-2 border rounded mb-2" value={buyerWhatsApp} onChange={e=>setBuyerWhatsApp(e.target.value)} />

              <label className="block text-sm">Cantidad de boletos</label>
              <input type="number" min={1} max={1000} className="w-full p-2 border rounded mb-2" value={qty} onChange={e=>setQty(Number(e.target.value))} />

              <p className="text-sm mb-2">Precio por boleto: <strong>{formatCurrency(state.settings.price)}</strong></p>
              <p className="text-sm mb-4">Total: <strong>{formatCurrency(qty * state.settings.price)}</strong></p>

              <div className="flex gap-2">
                <button className="px-4 py-2 bg-green-600 text-white rounded" onClick={buyTickets}>Comprar y notificar por WhatsApp</button>
                <button className="px-4 py-2 border rounded" onClick={()=>{setBuyerWhatsApp(''); setBuyerName(''); setQty(1); setMessage('');}}>Limpiar</button>
              </div>

              {message && <p className="mt-3 text-sm text-indigo-600">{message}</p>}

              <hr className="my-4" />
              <h3 className="font-semibold">Condiciones</h3>
              <ul className="text-sm list-disc ml-5">
                <li>El premio corresponde al {state.settings.prizePct}% de la venta total (valor aproximado).</li>
                <li>En demo, los pagos se registran manualmente en admin. En producción usar pasarela de pagos.</li>
                <li>Los pagos no son reembolsables.</li>
                <li>El premio se envía a cualquier parte de la república (gastos de envío a definir).</li>
              </ul>
            </section>

            <aside className="bg-white p-4 rounded shadow">
              <h2 className="text-xl font-semibold mb-2">Estadísticas</h2>
              <p>Boletos vendidos: <strong>{soldCount.toLocaleString()}</strong></p>
              <p>Boletos restantes: <strong>{remaining.toLocaleString()}</strong></p>
              <p>Recaudación estimada: <strong>{formatCurrency(soldCount * state.settings.price)}</strong></p>

              <div className="mt-4">
                <h3 className="font-semibold">Busca tus compras</h3>
                <input placeholder="filtrar por WhatsApp o nombre" className="w-full p-2 border rounded mt-2" value={filter} onChange={e=>setFilter(e.target.value)} />
                <div className="mt-3 max-h-56 overflow-auto">
                  {state.purchases.filter(p => !filter || (p.whatsapp||"").includes(filter) || (p.name||"").toLowerCase().includes(filter.toLowerCase())).map(p => (
                    <div key={p.id} className="border-b py-2">
                      <div className="flex items-center justify-between">
                        <div>
                          <div className="text-sm font-medium">{p.name || p.whatsapp}</div>
                          <div className="text-xs text-gray-500">{p.tickets.slice(0,5).join(', ')}{p.tickets.length>5?`... (+${p.tickets.length-5})`:''}</div>
                        </div>
                        <div className="text-sm">{p.paid ? <span className="text-green-600">Pagado</span> : <span className="text-yellow-600">Pendiente</span>}</div>
                      </div>
                    </div>
                  ))}
                </div>
              </div>
            </aside>
          </main>
        )}

        {route.startsWith(ADMIN_PATH) && (
          <section className="bg-white p-4 rounded shadow">
            {!route.includes('#logged') ? (
              <div>
                <h2 className="text-xl font-semibold mb-2">Admin — Iniciar sesión</h2>
                <p className="text-sm text-gray-600 mb-2">Ingrese la contraseña para acceder al panel de administración.</p>
                <input type="password" className="w-full p-2 border rounded mb-2" value={adminPassword} onChange={e=>setAdminPassword(e.target.value)} />
                <div className="flex gap-2">
                  <button className="px-4 py-2 bg-indigo-600 text-white rounded" onClick={adminLogin}>Entrar</button>
                  <button className="px-4 py-2 border rounded" onClick={()=>{setAdminPassword(ADMIN_PASSWORD_DEFAULT); alert('Se llenó la contraseña por defecto en demo.');}}>Autocompletar (demo)</button>
                </div>
              </div>
            ) : (
              <div>
                <h2 className="text-xl font-semibold mb-2">Panel de administración</h2>
                <div className="grid md:grid-cols-3 gap-4 mb-4">
                  <div className="p-3 border rounded">
                    <div className="text-sm text-gray-500">Boletos vendidos</div>
                    <div className="text-2xl font-bold">{soldCount.toLocaleString()}</div>
                  </div>
                  <div className="p-3 border rounded">
                    <div className="text-sm text-gray-500">Recaudación estimada</div>
                    <div className="text-2xl font-bold">{formatCurrency(soldCount * state.settings.price)}</div>
                  </div>
                  <div className="p-3 border rounded">
                    <div className="text-sm text-gray-500">Porcentaje premio</div>
                    <div className="flex items-center gap-2 mt-2">
                      <input type="number" min={1} max={100} value={state.settings.prizePct} onChange={e => setState({...state, settings:{...state.settings, prizePct: Number(e.target.value)}})} className="p-2 border rounded w-24" />
                      <button className="px-3 py-1 border rounded" onClick={()=>saveState(state)}>Guardar</button>
                    </div>
                  </div>
                </div>

                <div className="mb-4 flex gap-2">
                  <button className="px-3 py-2 bg-red-600 text-white rounded" onClick={drawWinner}>Realizar sorteo</button>
                  <button className="px-3 py-2 border rounded" onClick={exportSalesCSV}>Exportar ventas (CSV)</button>
                  <button className="px-3 py-2 border rounded" onClick={()=>{ if(confirm('Borrar todo el estado local (ventas y boletos)?')) { localStorage.removeItem(STORAGE_KEY); setState(makeInitialState()); alert('Estado reiniciado.'); } }}>Reiniciar demo</button>
                </div>

                <div className="max-h-96 overflow-auto border rounded p-2">
                  {state.purchases.length === 0 ? <p className="text-sm">No hay ventas aún.</p> : (
                    state.purchases.map(p => (
                      <div key={p.id} className="p-2 border-b flex items-start justify-between">
                        <div>
                          <div className="font-medium">{p.name || p.whatsapp} <span className="text-xs text-gray-400">({p.whatsapp})</span></div>
                          <div className="text-xs text-gray-600">{p.tickets.join(', ')}</div>
                          <div className="text-xs mt-1">{p.qty} boletos · {formatCurrency(p.totalPrice)} · {p.createdAt}</div>
                        </div>
                        <div className="flex flex-col items-end gap-2">
                          <div>{p.paid ? <span className="text-green-600 font-semibold">Pagado</span> : <button className="px-2 py-1 bg-green-600 text-white rounded text-sm" onClick={()=>markPaid(p.id)}>Marcar pagado</button>}</n          </div>
                          <div><button className="px-2 py-1 border rounded text-sm" onClick={()=>refundPurchase(p.id)}>Liberar boletos</button></div>
                        </div>
                      </div>
                    ))
                  )}
                </div>

              </div>
            )}
          </section>
        )}

        <footer className="mt-8 text-sm text-gray-500">Demo sin integración de pagos ni envío real de WhatsApp. Para producción: backend (Node/Java/PHP), base de datos (MySQL/Postgres), pasarela de pagos y proveedor de mensajes (Twilio, Meta API, etc.).</footer>
      </div>
    </div>
  );
}
