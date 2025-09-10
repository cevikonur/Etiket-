import React, { useEffect, useMemo, useRef, useState } from "react"; import { motion, AnimatePresence } from "framer-motion"; import { Button } from "@/components/ui/button"; import { Input } from "@/components/ui/input"; import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card"; import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogFooter } from "@/components/ui/dialog"; import { Tabs, TabsList, TabsTrigger, TabsContent } from "@/components/ui/tabs"; import { Label } from "@/components/ui/label"; import { Textarea } from "@/components/ui/textarea"; import { Select, SelectTrigger, SelectValue, SelectContent, SelectItem } from "@/components/ui/select"; import { Badge } from "@/components/ui/badge"; import { Trash2, ShoppingCart, Upload, Check, Minus, Plus, Download, Save, PackageSearch } from "lucide-react";

function formatCurrency(value, currency = "TRY") { try { return new Intl.NumberFormat("tr-TR", { style: "currency", currency }).format(Number(value || 0)); } catch { return ${value} ${currency}; } }

const STORAGE_KEYS = { CATALOG: "etiketapp_catalog_v1", CART: "etiketapp_cart_v1", ORDERS: "etiketapp_orders_v1", SETTINGS: "etiketapp_settings_v1", };

export default function EtiketSiparisApp() { const [catalog, setCatalog] = useState([]); // [{id,name,price,img,dataUrl}] const [cart, setCart] = useState({}); // {id: qty} const [currency, setCurrency] = useState("TRY"); const [customer, setCustomer] = useState({ name: "", phone: "", company: "", notes: "" }); const [orders, setOrders] = useState([]); const [activeTab, setActiveTab] = useState("siparis"); const fileInputRef = useRef(null); const [quickPrice, setQuickPrice] = useState(0); const [showBulkPriceDialog, setShowBulkPriceDialog] = useState(false); const [pendingFiles, setPendingFiles] = useState([]);

// Load persisted state useEffect(() => { const cat = localStorage.getItem(STORAGE_KEYS.CATALOG); const crt = localStorage.getItem(STORAGE_KEYS.CART); const ord = localStorage.getItem(STORAGE_KEYS.ORDERS); const stg = localStorage.getItem(STORAGE_KEYS.SETTINGS); if (cat) setCatalog(JSON.parse(cat)); if (crt) setCart(JSON.parse(crt)); if (ord) setOrders(JSON.parse(ord)); if (stg) { const s = JSON.parse(stg); if (s.currency) setCurrency(s.currency); if (s.customer) setCustomer(s.customer); } }, []);

// Persist changes automatically (arka planda kayıt) useEffect(() => { localStorage.setItem(STORAGE_KEYS.CATALOG, JSON.stringify(catalog)); }, [catalog]);

useEffect(() => { localStorage.setItem(STORAGE_KEYS.CART, JSON.stringify(cart)); }, [cart]);

useEffect(() => { localStorage.setItem( STORAGE_KEYS.SETTINGS, JSON.stringify({ currency, customer }) ); }, [currency, customer]);

const cartList = useMemo(() => { return Object.entries(cart) .map(([id, qty]) => { const item = catalog.find((c) => c.id === id); if (!item) return null; const total = Number(item.price || 0) * Number(qty || 0); return { ...item, qty, lineTotal: total }; }) .filter(Boolean); }, [cart, catalog]);

const grandTotal = useMemo(() => cartList.reduce((sum, it) => sum + it.lineTotal, 0), [cartList]);

function handleFilesSelected(files) { const arr = Array.from(files || []); if (!arr.length) return; setPendingFiles(arr); setShowBulkPriceDialog(true); }

async function readFileAsDataUrl(file) { return new Promise((resolve, reject) => { const reader = new FileReader(); reader.onload = () => resolve(reader.result); reader.onerror = reject; reader.readAsDataURL(file); }); }

async function confirmBulkAdd() { const newItems = []; for (const file of pendingFiles) { const dataUrl = await readFileAsDataUrl(file); newItems.push({ id: crypto.randomUUID(), name: file.name.replace(/.[^.]+$/, ""), price: Number(quickPrice || 0), img: dataUrl, createdAt: Date.now(), }); } setCatalog((prev) => [...newItems, ...prev]); setShowBulkPriceDialog(false); setPendingFiles([]); setQuickPrice(0); }

function addToCart(id) { setCart((prev) => ({ ...prev, [id]: (prev[id] || 0) + 1 })); }

function removeFromCart(id) { setCart((prev) => { const { [id]: _, ...rest } = prev; return rest; }); }

function setQty(id, qty) { setCart((prev) => ({ ...prev, [id]: Math.max(0, Number(qty || 0)) })); }

function inc(id) { setCart((p) => ({ ...p, [id]: (p[id] || 0) + 1 })); } function dec(id) { setCart((p) => ({ ...p, [id]: Math.max(0, (p[id] || 0) - 1) })); }

function updatePrice(id, price) { setCatalog((prev) => prev.map((it) => (it.id === id ? { ...it, price: Number(price || 0) } : it))); }

function deleteItem(id) { setCatalog((prev) => prev.filter((it) => it.id !== id)); setCart((prev) => { const cp = { ...prev }; delete cp[id]; return cp; }); }

function clearCart() { setCart({}); }

function createOrder() { if (!cartList.length) return alert("Sepet boş"); const order = { id: crypto.randomUUID(), customer, currency, items: cartList.map(({ id, name, price, qty }) => ({ id, name, price, qty })), total: grandTotal, createdAt: Date.now(), }; const next = [order, ...orders]; setOrders(next); localStorage.setItem(STORAGE_KEYS.ORDERS, JSON.stringify(next)); clearCart(); setActiveTab("siparisler"); }

function downloadOrder(order) { const blob = new Blob([JSON.stringify(order, null, 2)], { type: "application/json" }); const url = URL.createObjectURL(blob); const a = document.createElement("a"); a.href = url; a.download = siparis_${order.id}.json; a.click(); URL.revokeObjectURL(url); }

function printOrder(order) { const w = window.open("", "_blank"); if (!w) return; const rows = order.items .map((it) => <tr><td>${it.name}</td><td>${it.qty}</td><td>${formatCurrency(it.price, order.currency)}</td><td>${formatCurrency(it.price * it.qty, order.currency)}</td></tr>) .join(""); w.document.write(<html><head><title>Sipariş ${order.id}</title> <style>body{font-family:sans-serif;padding:24px} table{width:100%;border-collapse:collapse} td,th{border:1px solid #ddd;padding:8px} th{background:#f6f6f6;text-align:left}</style> </head><body> <h2>Sipariş #${order.id}</h2> <p><strong>Müşteri:</strong> ${order.customer.name || "-"} | <strong>Telefon:</strong> ${order.customer.phone || "-"} | <strong>Firma:</strong> ${order.customer.company || "-"}</p> <p><strong>Tarih:</strong> ${new Date(order.createdAt).toLocaleString("tr-TR")}</p> <table><thead><tr><th>Ürün</th><th>Adet</th><th>Birim</th><th>Tutar</th></tr></thead><tbody>${rows}</tbody></table> <h3>GENEL TOPLAM: ${formatCurrency(order.total, order.currency)}</h3> <p><em>Notlar:</em> ${order.customer.notes || "-"}</p> </body></html>); w.document.close(); w.focus(); w.print(); }

const totalCatalogValue = useMemo(() => catalog.reduce((s, it) => s + Number(it.price || 0), 0), [catalog]);

return ( <div className="min-h-screen bg-gradient-to-b from-white to-slate-50 p-4 md:p-8"> <div className="mx-auto max-w-7xl"> <header className="flex flex-col md:flex-row items-start md:items-center justify-between gap-4 mb-6"> <div> <h1 className="text-2xl md:text-3xl font-semibold tracking-tight">Asansör Etiket Sipariş Uygulaması</h1> <p className="text-slate-500">PNG/JPEG etiket yükle · Fiyat belirle · Sepet ve Adet · Otomatik kayıt</p> </div> <div className="flex items-center gap-2"> <Select value={currency} onValueChange={(v) => setCurrency(v)}> <SelectTrigger className="w-[120px]"><SelectValue placeholder="Para Birimi" /></SelectTrigger> <SelectContent> <SelectItem value="TRY">TRY</SelectItem> <SelectItem value="USD">USD</SelectItem> <SelectItem value="EUR">EUR</SelectItem> </SelectContent> </Select> <Button variant="outline" onClick={() => fileInputRef.current?.click()}> <Upload className="w-4 h-4 mr-2"/> Etiket Yükle </Button> <input ref={fileInputRef} type="file" accept="image/png,image/jpeg" multiple className="hidden" onChange={(e) => handleFilesSelected(e.target.files)} /> </div> </header>

<Tabs value={activeTab} onValueChange={setActiveTab}>
      <TabsList className="grid grid-cols-3 w-full md:w-[520px]">
        <TabsTrigger value="siparis" className="flex items-center gap-2"><ShoppingCart className="w-4 h-4"/> Sipariş</TabsTrigger>
        <TabsTrigger value="urunler" className="flex items-center gap-2"><PackageSearch className="w-4 h-4"/> Ürünler</TabsTrigger>
        <TabsTrigger value="siparisler" className="flex items-center gap-2"><Save className="w-4 h-4"/> Kayıtlı Siparişler</TabsTrigger>
      </TabsList>

      <TabsContent value="siparis" className="mt-6 grid grid-cols-1 lg:grid-cols-3 gap-6">
        {/* Catalog Grid */}
        <div className="lg:col-span-2">
          <div className="flex items-center justify-between mb-3">
            <h2 className="text-xl font-semibold">Etiket Kataloğu</h2>
            <Badge variant="secondary">Toplam Ürün: {catalog.length}</Badge>
          </div>

          {!catalog.length ? (
            <Card className="border-dashed">
              <CardContent className="p-8 text-center">
                <Upload className="w-10 h-10 mx-auto mb-3"/>
                <p className="text-slate-600">Başlamak için PNG/JPEG etiket görsellerini yükleyin.</p>
              </CardContent>
            </Card>
          ) : (
            <div className="grid sm:grid-cols-2 xl:grid-cols-3 gap-4">
              {catalog.map((it) => {
                const selected = !!cart[it.id] && cart[it.id] > 0;
                return (
                  <motion.div key={it.id} layout initial={{ opacity: 0, scale: 0.98 }} animate={{ opacity: 1, scale: 1 }}>
                    <Card className={`overflow-hidden transition-colors ${selected ? "bg-green-50 border-green-300" : ""}`}>
                      <CardHeader className="pb-2">
                        <CardTitle className="text-base flex items-center justify-between">
                          <span className="truncate" title={it.name}>{it.name}</span>
                          {selected && <Badge className="ml-2 bg-green-600">Seçildi</Badge>}
                        </CardTitle>
                      </CardHeader>
                      <CardContent className="space-y-3">
                        <img src={it.img} alt={it.name} className="w-full aspect-square object-contain bg-white rounded-xl border p-2"/>
                        <div className="flex items-center gap-2">
                          <Label className="text-sm">Fiyat:</Label>
                          <Input type="number" min={0} step="0.01" value={it.price ?? ""} onChange={(e) => updatePrice(it.id, e.target.value)} className="h-9"/>
                          <span className="text-sm text-slate-600">{formatCurrency(it.price, currency)}</span>
                        </div>
                        <div className="flex items-center justify-between gap-2">
                          {!selected ? (
                            <Button className="w-full" onClick={() => addToCart(it.id)}>
                              <Check className="w-4 h-4 mr-2"/> Sepete Ekle
                            </Button>
                          ) : (
                            <div className="flex items-center w-full gap-2">
                              <Button size="icon" variant="outline" onClick={() => dec(it.id)}><Minus className="w-4 h-4"/></Button>
                              <Input type="number" className="w-20 text-center" value={cart[it.id]} min={0} onChange={(e) => setQty(it.id, e.target.value)} />
                              <Button size="icon" variant="outline" onClick={() => inc(it.id)}><Plus className="w-4 h-4"/></Button>
                              <Button variant="destructive" className="ml-auto" onClick={() => removeFromCart(it.id)}><Trash2 className="w-4 h-4 mr-2"/> Kaldır</Button>
                            </div>
                          )}
                        </div>
                      </CardContent>
                    </Card>
                  </motion.div>
                );
              })}
            </div>
          )}
        </div>

        {/* Cart */}
        <div>
          <Card className="sticky top-6">
            <CardHeader>
              <CardTitle className="flex items-center gap-2"><ShoppingCart className="w-5 h-5"/> Sepet</CardTitle>
            </CardHeader>
            <CardContent className="space-y-4">
              <div className="space-y-2 max-h-72 overflow-auto pr-1">
                {!cartList.length && <p className="text-slate-500">Henüz ürün seçmediniz.</p>}
                {cartList.map((it) => (
                  <div key={it.id} className="flex items-center gap-3 p-2 rounded-xl border bg-white">
                    <img src={it.img} alt={it.name} className="w-12 h-12 object-contain rounded-md border"/>
                    <div className="flex-1 min-w-0">
                      <div className="flex items-center justify-between">
                        <span className="font-medium truncate" title={it.name}>{it.name}</span>
                        <Button size="icon" variant="ghost" onClick={() => removeFromCart(it.id)}><Trash2 className="w-4 h-4"/></Button>
                      </div>
                      <div className="flex items-center gap-2 mt-1">
                        <Button size="icon" variant="outline" onClick={() => dec(it.id)}><Minus className="w-4 h-4"/></Button>
                        <Input type="number" className="w-20 text-center" value={it.qty} min={0} onChange={(e) => setQty(it.id, e.target.value)} />
                        <Button size="icon" variant="outline" onClick={() => inc(it.id)}><Plus className="w-4 h-4"/></Button>
                        <span className="ml-auto text-sm">{formatCurrency(it.price, currency)} × {it.qty} = <strong>{formatCurrency(it.lineTotal, currency)}</strong></span>
                      </div>
                    </div>
                  </div>
                ))}
              </div>

              <div className="border-t pt-3 space-y-2">
                <div className="flex items-center justify-between"><span>Ara Toplam</span><span>{formatCurrency(grandTotal, currency)}</span></div>
                <div className="flex items-center justify-between font-semibold text-lg"><span>Genel Toplam</span><span>{formatCurrency(grandTotal, currency)}</span></div>
                <div className="flex items-center justify-end gap-2">
                  <Button variant="outline" onClick={clearCart}>Sepeti Temizle</Button>
                  <Button onClick={createOrder}><Check className="w-4 h-4 mr-2"/> Sipariş Oluştur</Button>
                </div>
              </div>

              <div className="border-t pt-4 space-y-3">
                <h3 className="font-semibold">Müşteri Bilgileri</h3>
                <div className="grid grid-cols-1 md:grid-cols-2 gap-2">
                  <div>
                    <Label>Ad Soyad</Label>
                    <Input value={customer.name} onChange={(e) => setCustomer({ ...customer, name: e.target.value })} />
                  </div>
                  <div>
                    <Label>Telefon</Label>
                    <Input value={customer.phone} onChange={(e) => setCustomer({ ...customer, phone: e.target.value })} />
                  </div>
                  <div>
                    <Label>Firma</Label>
                    <Input value={customer.company} onChange={(e) => setCustomer({ ...customer, company: e.target.value })} />
                  </div>
                  <div className="md:col-span-2">
                    <Label>Notlar</Label>
                    <Textarea value={customer.notes} onChange={(e) => setCustomer({ ...customer, notes: e.target.value })} />
                  </div>
                </div>
              </div>
            </CardContent>
          </Card>
        </div>
      </TabsContent>

      <TabsContent value="urunler" className="mt-6 space-y-4">
        <Card>
          <CardHeader>
            <CardTitle>Toplu Yükleme</CardTitle>
          </CardHeader>
          <CardContent className="space-y-4">
            <div className="flex flex-col md:flex-row items-start md:items-center gap-3">
              <div className="flex items-center gap-2">
                <Button variant="outline" onClick={() => fileInputRef.current?.click()}><Upload className="w-4 h-4 mr-2"/> Görsel Seç</Button>
                <span className="text-slate-500 text-sm">PNG veya JPEG, birden fazla seçebilirsiniz.</span>
              </div>
              <div className="flex items-center gap-2">
                <Label>Varsayılan Fiyat</Label>
                <Input type="number" value={quickPrice} onChange={(e) => setQuickPrice(e.target.value)} className="w-40"/>
                <Badge variant="secondary">Örn: {formatCurrency(quickPrice || 0, currency)}</Badge>
              </div>
            </div>
            <div className="text-sm text-slate-600">Yüklenen her yeni etikete bu fiyat atanır. Daha sonra tek tek değiştirebilirsiniz.</div>
            <div className="flex items-center gap-2">
              <Badge>Katalog Değeri ~ {formatCurrency(totalCatalogValue, currency)}</Badge>
            </div>
          </CardContent>
        </Card>

        <div className="grid sm:grid-cols-2 xl:grid-cols-3 gap-4">
          {catalog.map((it) => (
            <Card key={it.id} className="overflow-hidden">
              <CardHeader className="pb-2 flex-row items-center justify-between">
                <CardTitle className="text-base truncate" title={it.name}>{it.name}</CardTitle>
                <Button size="icon" variant="ghost" onClick={() => deleteItem(it.id)}><Trash2 className="w-4 h-4"/></Button>
              </CardHeader>
              <CardContent className="space-y-3">
                <img src={it.img} alt={it.name} className="w-full aspect-square object-contain bg-white rounded-xl border p-2"/>
                <div className="flex items-center gap-2">
                  <Label>Fiyat</Label>
                  <Input type="number" min={0} step="0.01" value={it.price ?? ""} onChange={(e) => updatePrice(it.id, e.target.value)} className="h-9"/>
                  <span className="text-sm text-slate-600">{formatCurrency(it.price, currency)}</span>
                </div>
              </CardContent>
            </Card>
          ))}
        </div>
      </TabsContent>

      <TabsContent value="siparisler" className="mt-6">
        <Card>
          <CardHeader>
            <CardTitle>Kayıtlı Siparişler</CardTitle>
          </CardHeader>
          <CardContent className="space-y-4">
            {!orders.length ? (
              <p className="text-slate-500">Henüz kayıtlı sipariş yok.</p>
            ) : (
              <div className="space-y-3">
                {orders.map((o) => (
                  <motion.div key={o.id} layout className="p-3 rounded-xl border bg-white">
                    <div className="flex flex-col md:flex-row md:items-center gap-2 md:gap-4">
                      <div className="flex-1">
                        <div className="flex items-center gap-2">
                          <Badge variant="outline">#{o.id.slice(0, 8)}</Badge>
                          <span className="text-slate-600">{new Date(o.createdAt).toLocaleString("tr-TR")}</span>
                        </div>
                        <div className="text-sm text-slate-700 mt-1">
                          Müşteri: <strong>{o.customer?.name || "-"}</strong> • Toplam: <strong>{formatCurrency(o.total, o.currency)}</strong> • Kalem: {o.items.length}
                        </div>
                      </div>
                      <div className="flex gap-2 ml-auto">
                        <Button variant="outline" onClick={() => downloadOrder(o)}><Download className="w-4 h-4 mr-2"/> JSON</Button>
                        <Button onClick={() => printOrder(o)}><Save className="w-4 h-4 mr-2"/> Yazdır</Button>
                      </div>
                    </div>
                    <div className="mt-3 grid md:grid-cols-2 lg:grid-cols-3 gap-2">
                      {o.items.map((it) => (
                        <div key={it.id} className="text-sm p-2 rounded-lg border bg-slate-50 flex items-center gap-2">
                          <span className="font-medium truncate">{it.name}</span>
                          <span className="ml-auto">{it.qty} × {formatCurrency(it.price, o.currency)}</span>
                        </div>
                      ))}
                    </div>
                  </motion.div>
                ))}
              </div>
            )}
          </CardContent>
        </Card>
      </TabsContent>
    </Tabs>
  </div>

  {/* Bulk price dialog after file choose */}
  <Dialog open={showBulkPriceDialog} onOpenChange={setShowBulkPriceDialog}>
    <DialogContent>
      <DialogHeader>
        <DialogTitle>Toplu Yükleme – Varsayılan Fiyat</DialogTitle>
      </DialogHeader>
      <div className="space-y-3">
        <p className="text-slate-600">{pendingFiles.length} görsel seçildi. Bu görseller için başlangıç fiyatını girin. (Daha sonra ürünler sekmesinden değiştirebilirsiniz.)</p>
        <div className="flex items-center gap-2">
          <Label>Fiyat</Label>
          <Input type="number" value={quickPrice} onChange={(e) => setQuickPrice(e.target.value)} className="w-40"/>
          <Badge variant="secondary">{formatCurrency(quickPrice || 0, currency)}</Badge>
        </div>
      </div>
      <DialogFooter>
        <Button variant="outline" onClick={() => setShowBulkPriceDialog(false)}>İptal</Button>
        <Button onClick={confirmBulkAdd}><Check className="w-4 h-4 mr-2"/> Ekle</Button>
      </DialogFooter>
    </DialogContent>
  </Dialog>
</div>

); }

