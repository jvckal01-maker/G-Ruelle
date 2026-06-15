# G-Ruelle
<!DOCTYPE html>
<html lang="fr">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>LA RUELLE — Suivi de Commandes</title>
  <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
  <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
  <script src="https://cdn.tailwindcss.com"></script>
  <style>
    body { margin: 0; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', sans-serif; }
    .kpi-card { transition: transform 0.2s, box-shadow 0.2s; }
    .kpi-card:hover { transform: translateY(-2px); box-shadow: 0 20px 25px rgba(0,0,0,0.3); }
    input::-webkit-calendar-picker-indicator { cursor: pointer; }
  </style>
</head>
<body class="bg-gradient-to-br from-gray-900 via-blue-900 to-gray-900 min-h-screen">
  <div id="root"></div>
  <script type="text/babel">
    const { useState, useEffect } = React;

    function LaRuelleDashboard() {
      const [tab, setTab] = useState('dashboard');
      const [orders, setOrders] = useState([]);
      const [stock, setStock] = useState([]);
      const [showOrderForm, setShowOrderForm] = useState(false);
      const [editingOrder, setEditingOrder] = useState(null);
      const [searchClient, setSearchClient] = useState('');

      const defaultOrders = [
        { id: 'CMD-001', date: '2026-06-13', time: '08:30', client: 'Konan Koffi', product: 'Pochette téléphone', qty: 3, price: 2500, status: 'Livré', paid: 7500, mode: 'Orange Money' },
        { id: 'CMD-002', date: '2026-06-13', time: '09:15', client: 'Aya Bamba', product: 'Coque iPhone 14', qty: 1, price: 8000, status: 'En cours', paid: 5000, mode: 'Mobile Money' },
        { id: 'CMD-003', date: '2026-06-13', time: '10:00', client: 'Djiby Traoré', product: 'Chargeur rapide', qty: 2, price: 3500, status: 'En attente', paid: 0, mode: '' },
        { id: 'CMD-004', date: '2026-06-13', time: '11:45', client: 'Marie Kouassi', product: 'Écouteurs Bluetooth', qty: 1, price: 12000, status: 'Livré', paid: 12000, mode: 'Espèces' },
        { id: 'CMD-005', date: '2026-06-13', time: '14:20', client: 'Seydou Fall', product: 'Powerbank 20000mAh', qty: 1, price: 9500, status: 'Annulé', paid: 0, mode: '' },
        { id: 'CMD-006', date: '2026-06-13', time: '15:00', client: 'Binta Diallo', product: 'Film protection', qty: 4, price: 1500, status: 'Livré', paid: 6000, mode: 'Wave' },
      ];

      const defaultStock = [
        { ref: 'REF-001', product: 'Pochette téléphone', category: 'Accessoires', initial: 50, minStock: 10, buyPrice: 1500, sellPrice: 2500 },
        { ref: 'REF-002', product: 'Coque iPhone 14', category: 'Coques', initial: 30, minStock: 5, buyPrice: 4000, sellPrice: 8000 },
        { ref: 'REF-003', product: 'Chargeur rapide', category: 'Chargeurs', initial: 40, minStock: 10, buyPrice: 1800, sellPrice: 3500 },
        { ref: 'REF-004', product: 'Écouteurs BT', category: 'Audio', initial: 20, minStock: 5, buyPrice: 6000, sellPrice: 12000 },
        { ref: 'REF-005', product: 'Powerbank 20000mAh', category: 'Chargeurs', initial: 15, minStock: 3, buyPrice: 5000, sellPrice: 9500 },
        { ref: 'REF-006', product: 'Film protection', category: 'Accessoires', initial: 100, minStock: 20, buyPrice: 400, sellPrice: 1500 },
      ];

      useEffect(() => {
        const saved = localStorage.getItem('laRuelleMecanix');
        if (saved) {
          try {
            const { orders: o, stock: s } = JSON.parse(saved);
            setOrders(o || defaultOrders);
            setStock(s || defaultStock);
          } catch {
            setOrders(defaultOrders);
            setStock(defaultStock);
          }
        } else {
          setOrders(defaultOrders);
          setStock(defaultStock);
        }
      }, []);

      useEffect(() => {
        localStorage.setItem('laRuelleMecanix', JSON.stringify({ orders, stock }));
      }, [orders, stock]);

      const getTotal = (qty, price) => qty * price;
      const getSoldQty = (productName) => orders
        .filter(o => o.product === productName && o.status !== 'Annulé')
        .reduce((sum, o) => sum + o.qty, 0);
      const getCurrentStock = (productName) => {
        const s = stock.find(st => st.product === productName);
        return s ? s.initial - getSoldQty(productName) : 0;
      };

      const monthOrders = orders.filter(o => {
        const d = new Date(o.date);
        const now = new Date();
        return d.getMonth() === now.getMonth() && d.getFullYear() === now.getFullYear();
      });

      const kpis = {
        totalOrders: monthOrders.length,
        totalCA: monthOrders.reduce((s, o) => s + getTotal(o.qty, o.price), 0),
        delivered: monthOrders.filter(o => o.status === 'Livré').length,
        pending: monthOrders.filter(o => ['En attente', 'En cours'].includes(o.status)).length,
        collected: monthOrders.reduce((s, o) => s + (o.status === 'Livré' ? o.paid : 0), 0),
        remaining: monthOrders.reduce((s, o) => o.status !== 'Annulé' ? s + (getTotal(o.qty, o.price) - o.paid) : s, 0),
        lowStock: stock.filter(s => getCurrentStock(s.product) <= s.minStock).length,
        cancelled: monthOrders.filter(o => o.status === 'Annulé').length,
      };

      const dayNames = ['Lun', 'Mar', 'Mer', 'Jeu', 'Ven', 'Sam', 'Dim'];
      const daysSales = Array(7).fill(0);
      monthOrders.forEach(o => {
        if (o.status !== 'Annulé') {
          const d = new Date(o.date);
          const dow = (d.getDay() + 6) % 7;
          daysSales[dow] += getTotal(o.qty, o.price);
        }
      });

      const hourBands = ['08-10', '10-12', '12-14', '14-16', '16-18', '18-20'];
      const hourSales = Array(6).fill(0);
      monthOrders.forEach(o => {
        if (o.status !== 'Annulé') {
          const hour = parseInt(o.time.split(':')[0]);
          const idx = Math.floor((hour - 8) / 2);
          if (idx >= 0 && idx < 6) hourSales[idx] += getTotal(o.qty, o.price);
        }
      });

      const maxDay = Math.max(...daysSales, 1);
      const maxHour = Math.max(...hourSales, 1);

      const handleAddOrder = (formData) => {
        if (editingOrder) {
          setOrders(orders.map(o => o.id === editingOrder.id ? { ...formData, id: editingOrder.id } : o));
          setEditingOrder(null);
        } else {
          const newId = `CMD-${String(orders.filter(o => o.id.startsWith('CMD-')).length + 1).padStart(3, '0')}`;
          setOrders([...orders, { ...formData, id: newId }]);
        }
        setShowOrderForm(false);
      };

      const handleDeleteOrder = (id) => {
        if (confirm('Supprimer cette commande ?')) {
          setOrders(orders.filter(o => o.id !== id));
        }
      };

      const filteredOrders = orders.filter(o => !searchClient || o.client.toLowerCase().includes(searchClient.toLowerCase()));

      return (
        <div className="min-h-screen bg-gradient-to-br from-gray-900 via-blue-900 to-gray-900">
          {/* Header */}
          <div className="sticky top-0 z-40 border-b border-gray-700 bg-gray-900/95 backdrop-blur">
            <div className="max-w-7xl mx-auto px-4 py-4">
              <div className="flex items-center justify-between mb-4">
                <div>
                  <h1 className="text-3xl font-bold text-white">📦 LA RUELLE</h1>
                  <p className="text-sm text-gray-400 mt-1">Suivi intelligent des commandes et du stock</p>
                </div>
              </div>

              <div className="flex gap-2 flex-wrap">
                {[
                  { id: 'dashboard', label: '📊 Tableau de bord' },
                  { id: 'orders', label: '📦 Commandes' },
                  { id: 'stock', label: '📋 Stock' },
                ].map(t => (
                  <button
                    key={t.id}
                    onClick={() => setTab(t.id)}
                    className={`px-4 py-2 rounded-lg font-medium transition-all ${
                      tab === t.id
                        ? 'bg-blue-600 text-white shadow-lg shadow-blue-600/50'
                        : 'bg-gray-800 text-gray-300 hover:bg-gray-700'
                    }`}
                  >
                    {t.label}
                  </button>
                ))}
              </div>
            </div>
          </div>

          <div className="max-w-7xl mx-auto px-4 py-8">
            {/* DASHBOARD */}
            {tab === 'dashboard' && (
              <div className="space-y-6">
                {/* KPI Grid */}
                <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
                  {[
                    { label: '💰 CA du mois', value: kpis.totalCA, color: 'from-blue-600 to-blue-700' },
                    { label: '✅ Livrées', value: kpis.delivered, color: 'from-green-600 to-green-700' },
                    { label: '⏳ Pending', value: kpis.pending, color: 'from-orange-600 to-orange-700' },
                  ].map((kpi, i) => (
                    <div
                      key={i}
                      className={`kpi-card bg-gradient-to-br ${kpi.color} p-6 rounded-xl text-white shadow-xl`}
                    >
                      <div className="text-sm font-medium opacity-90">{kpi.label}</div>
                      <div className="text-4xl font-bold mt-2">
                        {kpi.value >= 1000 ? `${(kpi.value / 1000).toFixed(0)}K` : kpi.value}
                      </div>
                    </div>
                  ))}
                </div>

                {/* Secondary KPIs */}
                <div className="grid grid-cols-2 md:grid-cols-4 gap-3">
                  {[
                    { label: '💵 Encaissé', value: kpis.collected, color: 'from-indigo-600 to-indigo-700' },
                    { label: '🔴 À encaisser', value: kpis.remaining, color: 'from-red-600 to-red-700' },
                    { label: '⚠️ Stock bas', value: kpis.lowStock, color: 'from-purple-600 to-purple-700' },
                    { label: '❌ Annulées', value: kpis.cancelled, color: 'from-gray-600 to-gray-700' },
                  ].map((kpi, i) => (
                    <div
                      key={i}
                      className={`kpi-card bg-gradient-to-br ${kpi.color} p-4 rounded-lg text-white shadow-lg`}
                    >
                      <div className="text-xs font-medium opacity-90 mb-1">{kpi.label}</div>
                      <div className="text-2xl font-bold">{kpi.value >= 1000 ? `${(kpi.value / 1000).toFixed(0)}K` : kpi.value}</div>
                    </div>
                  ))}
                </div>

                {/* Best days & hours */}
                <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
                  {/* Best days */}
                  <div className="bg-white rounded-xl p-6 shadow-xl">
                    <h2 className="text-lg font-bold mb-4 text-gray-900">📅 Meilleurs jours</h2>
                    <div className="space-y-3">
                      {dayNames.map((day, i) => (
                        <div key={i} className="flex items-center gap-3">
                          <div className="w-12 font-bold text-gray-700">{day}</div>
                          <div className="flex-1 bg-gray-200 rounded-full h-8 overflow-hidden">
                            <div
                              className="bg-gradient-to-r from-blue-400 to-blue-600 h-full flex items-center justify-end pr-2 transition-all"
                              style={{ width: `${(daysSales[i] / maxDay) * 100}%` }}
                            >
                              {daysSales[i] > 0 && (
                                <span className="text-white text-xs font-bold">{(daysSales[i] / 1000).toFixed(0)}K</span>
                              )}
                            </div>
                          </div>
                        </div>
                      ))}
                    </div>
                  </div>

                  {/* Peak hours */}
                  <div className="bg-white rounded-xl p-6 shadow-xl">
                    <h2 className="text-lg font-bold mb-4 text-gray-900">⏰ Heures de pointe</h2>
                    <div className="space-y-3">
                      {hourBands.map((band, i) => (
                        <div key={i} className="flex items-center gap-3">
                          <div className="w-16 text-sm font-bold text-gray-700">{band}h</div>
                          <div className="flex-1 bg-gray-200 rounded-full h-8 overflow-hidden">
                            <div
                              className="bg-gradient-to-r from-orange-400 to-orange-600 h-full flex items-center justify-end pr-2 transition-all"
                              style={{ width: `${(hourSales[i] / maxHour) * 100}%` }}
                            >
                              {hourSales[i] > 0 && (
                                <span className="text-white text-xs font-bold">{(hourSales[i] / 1000).toFixed(0)}K</span>
                              )}
                            </div>
                          </div>
                        </div>
                      ))}
                    </div>
                  </div>
                </div>
              </div>
            )}

            {/* ORDERS */}
            {tab === 'orders' && (
              <div className="space-y-4">
                <div className="flex flex-col md:flex-row justify-between items-start md:items-center gap-4">
                  <div className="flex-1">
                    <h2 className="text-2xl font-bold text-white mb-2">Commandes</h2>
                    <input
                      type="text"
                      placeholder="Chercher par client..."
                      value={searchClient}
                      onChange={(e) => setSearchClient(e.target.value)}
                      className="w-full px-4 py-2 rounded-lg bg-gray-800 text-white placeholder-gray-500 focus:outline-none focus:ring-2 focus:ring-blue-500"
                    />
                  </div>
                  <button
                    onClick={() => { setEditingOrder(null); setShowOrderForm(true); }}
                    className="px-4 py-2 bg-blue-600 hover:bg-blue-700 text-white rounded-lg font-medium whitespace-nowrap"
                  >
                    ➕ Nouvelle
                  </button>
                </div>

                <div className="bg-white rounded-xl shadow-xl overflow-hidden">
                  <div className="overflow-x-auto">
                    <table className="w-full text-sm">
                      <thead className="bg-gray-900 text-white">
                        <tr>
                          <th className="px-4 py-3 text-left font-semibold">N°</th>
                          <th className="px-4 py-3 text-left font-semibold">Date</th>
                          <th className="px-4 py-3 text-left font-semibold">Client</th>
                          <th className="px-4 py-3 text-left font-semibold">Produit</th>
                          <th className="px-4 py-3 text-center font-semibold">Qté</th>
                          <th className="px-4 py-3 text-right font-semibold">Total</th>
                          <th className="px-4 py-3 text-center font-semibold">Statut</th>
                          <th className="px-4 py-3 text-right font-semibold">Reste</th>
                          <th className="px-4 py-3 text-center font-semibold">Action</th>
                        </tr>
                      </thead>
                      <tbody>
                        {filteredOrders.map((o, i) => {
                          const total = getTotal(o.qty, o.price);
                          const remaining = total - o.paid;
                          const statusStyle = {
                            'Livré': 'bg-green-100 text-green-800',
                            'En cours': 'bg-yellow-100 text-yellow-800',
                            'En attente': 'bg-orange-100 text-orange-800',
                            'Annulé': 'bg-red-100 text-red-800',
                          }[o.status] || '';

                          return (
                            <tr key={i} className={i % 2 ? 'bg-gray-50' : 'bg-white'}>
                              <td className="px-4 py-3 font-mono font-bold text-gray-600">{o.id}</td>
                              <td className="px-4 py-3 text-gray-600">{o.date}</td>
                              <td className="px-4 py-3">{o.client}</td>
                              <td className="px-4 py-3 text-gray-600">{o.product}</td>
                              <td className="px-4 py-3 text-center text-gray-600">{o.qty}</td>
                              <td className="px-4 py-3 text-right font-mono">{total.toLocaleString('fr-FR')}F</td>
                              <td className="px-4 py-3 text-center">
                                <span className={`px-2 py-1 rounded-full text-xs font-bold ${statusStyle}`}>
                                  {o.status}
                                </span>
                              </td>
                              <td className={`px-4 py-3 text-right font-mono font-bold ${remaining > 0 ? 'text-red-600' : 'text-green-600'}`}>
                                {remaining > 0 ? remaining.toLocaleString('fr-FR') : '✓'}F
                              </td>
                              <td className="px-4 py-3 text-center">
                                <div className="flex gap-2 justify-center">
                                  <button
                                    onClick={() => { setEditingOrder(o); setShowOrderForm(true); }}
                                    className="text-blue-600 hover:text-blue-800 p-1 hover:bg-blue-50 rounded"
                                  >
                                    ✎
                                  </button>
                                  <button
                                    onClick={() => handleDeleteOrder(o.id)}
                                    className="text-red-600 hover:text-red-800 p-1 hover:bg-red-50 rounded"
                                  >
                                    🗑
                                  </button>
                                </div>
                              </td>
                            </tr>
                          );
                        })}
                      </tbody>
                    </table>
                  </div>
                </div>
              </div>
            )}

            {/* STOCK */}
            {tab === 'stock' && (
              <div className="space-y-4">
                <h2 className="text-2xl font-bold text-white">Gestion du stock</h2>
                <div className="bg-white rounded-xl shadow-xl overflow-hidden">
                  <div className="overflow-x-auto">
                    <table className="w-full text-sm">
                      <thead className="bg-gray-900 text-white">
                        <tr>
                          <th className="px-4 py-3 text-left font-semibold">Ref</th>
                          <th className="px-4 py-3 text-left font-semibold">Produit</th>
                          <th className="px-4 py-3 text-left font-semibold">Catégorie</th>
                          <th className="px-4 py-3 text-center font-semibold">Init</th>
                          <th className="px-4 py-3 text-center font-semibold">Vendu</th>
                          <th className="px-4 py-3 text-center font-semibold">Actuel</th>
                          <th className="px-4 py-3 text-center font-semibold">Mini</th>
                          <th className="px-4 py-3 text-center font-semibold">Alerte</th>
                        </tr>
                      </thead>
                      <tbody>
                        {stock.map((s, i) => {
                          const sold = getSoldQty(s.product);
                          const current = getCurrentStock(s.product);
                          const isLow = current <= s.minStock;

                          return (
                            <tr key={i} className={i % 2 ? 'bg-gray-50' : 'bg-white'}>
                              <td className="px-4 py-3 font-mono font-bold text-gray-600">{s.ref}</td>
                              <td className="px-4 py-3">{s.product}</td>
                              <td className="px-4 py-3 text-gray-600 text-xs">{s.category}</td>
                              <td className="px-4 py-3 text-center text-gray-600">{s.initial}</td>
                              <td className="px-4 py-3 text-center text-orange-600 font-bold">{sold}</td>
                              <td className={`px-4 py-3 text-center font-bold ${isLow ? 'text-red-600' : 'text-green-600'}`}>
                                {current}
                              </td>
                              <td className="px-4 py-3 text-center text-gray-600">{s.minStock}</td>
                              <td className="px-4 py-3 text-center">
                                <span className={`px-2 py-1 rounded-full text-xs font-bold ${isLow ? 'bg-red-100 text-red-800' : 'bg-green-100 text-green-800'}`}>
                                  {isLow ? '⚠️ RÉAPPRO' : '✅ OK'}
                                </span>
                              </td>
                            </tr>
                          );
                        })}
                      </tbody>
                    </table>
                  </div>
                </div>
              </div>
            )}
          </div>

          {/* Modal */}
          {showOrderForm && (
            <OrderFormModal
              order={editingOrder}
              onSave={handleAddOrder}
              onClose={() => { setShowOrderForm(false); setEditingOrder(null); }}
              products={stock.map(s => s.product)}
            />
          )}
        </div>
      );
    }

    function OrderFormModal({ order, onSave, onClose, products }) {
      const [formData, setFormData] = React.useState(order || {
        date: new Date().toISOString().split('T')[0],
        time: '12:00',
        client: '',
        product: '',
        qty: 1,
        price: 0,
        status: 'En attente',
        paid: 0,
        mode: '',
      });

      const handleChange = (field, value) => {
        setFormData(prev => ({ ...prev, [field]: value }));
      };

      const handleSubmit = (e) => {
        e.preventDefault();
        if (formData.client && formData.product) {
          onSave(formData);
        }
      };

      return (
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4">
          <div className="bg-white rounded-xl max-w-2xl w-full p-8 shadow-2xl">
            <div className="flex justify-between items-center mb-6">
              <h2 className="text-2xl font-bold text-gray-900">{order ? '✏️ Modifier' : '✨ Nouvelle commande'}</h2>
              <button onClick={onClose} className="text-gray-500 hover:text-gray-700 text-2xl">✕</button>
            </div>

            <form onSubmit={handleSubmit} className="space-y-4">
              <div className="grid grid-cols-2 md:grid-cols-3 gap-4">
                <input
                  type="date"
                  value={formData.date}
                  onChange={(e) => handleChange('date', e.target.value)}
                  className="px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                  required
                />
                <input
                  type="time"
                  value={formData.time}
                  onChange={(e) => handleChange('time', e.target.value)}
                  className="px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                  required
                />
                <input
                  type="text"
                  placeholder="Client"
                  value={formData.client}
                  onChange={(e) => handleChange('client', e.target.value)}
                  className="px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 md:col-span-3"
                  required
                />
                <select
                  value={formData.product}
                  onChange={(e) => handleChange('product', e.target.value)}
                  className="px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 md:col-span-2"
                  required
                >
                  <option value="">Produit...</option>
                  {products.map(p => <option key={p} value={p}>{p}</option>)}
                </select>
                <input
                  type="number"
                  min="1"
                  placeholder="Qté"
                  value={formData.qty}
                  onChange={(e) => handleChange('qty', parseInt(e.target.value) || 1)}
                  className="px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                  required
                />
                <input
                  type="number"
                  min="0"
                  placeholder="Prix FCFA"
                  value={formData.price}
                  onChange={(e) => handleChange('price', parseInt(e.target.value) || 0)}
                  className="px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                  required
                />
                <input
                  type="number"
                  min="0"
                  placeholder="Payé FCFA"
                  value={formData.paid}
                  onChange={(e) => handleChange('paid', parseInt(e.target.value) || 0)}
                  className="px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                />
                <select
                  value={formData.status}
                  onChange={(e) => handleChange('status', e.target.value)}
                  className="px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
                >
                  {['En attente', 'En cours', 'Livré', 'Annulé'].map(s => (
                    <option key={s} value={s}>{s}</option>
                  ))}
                </select>
                <select
                  value={formData.mode}
                  onChange={(e) => handleChange('mode', e.target.value)}
                  className="px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500 md:col-span-2"
                >
                  <option value="">Mode paiement...</option>
                  {['Espèces', 'Orange Money', 'Mobile Money', 'Wave', 'Virement'].map(m => (
                    <option key={m} value={m}>{m}</option>
                  ))}
                </select>
              </div>

              <div className="flex gap-3 justify-end pt-4 border-t">
                <button
                  type="button"
                  onClick={onClose}
                  className="px-6 py-2 rounded-lg border border-gray-300 text-gray-700 hover:bg-gray-100 font-medium"
                >
                  Annuler
                </button>
                <button
                  type="submit"
                  className="px-6 py-2 rounded-lg bg-blue-600 text-white hover:bg-blue-700 font-medium"
                >
                  ✓ Enregistrer
                </button>
              </div>
            </form>
          </div>
        </div>
      );
    }

    ReactDOM.render(<LaRuelleDashboard />, document.getElementById('root'));
  </script>
</body>
</html>
