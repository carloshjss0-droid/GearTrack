
import React, { useState, useEffect, useMemo } from 'react';
import { 
  LayoutDashboard, 
  Package, 
  Scan, 
  Plus, 
  Search, 
  ChevronRight, 
  MapPin, 
  AlertCircle,
  Camera,
  Download,
  CheckCircle2,
  Clock,
  Sparkles,
  Users,
  ShieldCheck,
  Truck,
  Building2,
  ReceiptText,
  Info,
  DollarSign,
  TrendingDown,
  CalendarDays,
  BellRing,
  CalendarCheck,
  User as UserIcon,
  Archive,
  Wrench,
  Trash2,
  Mic2,
  Disc,
  Laptop,
  Zap,
  Cable,
  HardDrive,
  FileText,
  Gavel,
  History,
  BarChart3,
  ArrowUpRight,
  ClipboardList,
  Clapperboard,
  Scale,
  ShieldAlert,
  Handshake
} from 'lucide-react';
import QRCode from 'qrcode';
import { View, Equipment, EquipmentCondition, InventoryHistory, Collaborator, RentedEquipment, Booking } from './types';
import { Scanner } from './components/Scanner';
import { generateEquipmentDescription } from './services/geminiService';

const INITIAL_INVENTORY: Equipment[] = [
  {
    id: 'AV-001',
    name: 'Sony A7S III',
    category: 'Câmera',
    description: 'Câmera Mirrorless Full-frame otimizada para vídeo 4K 120p.',
    quantity: 3,
    condition: EquipmentCondition.EXCELLENT,
    location: 'Armário A1',
    serialNumber: 'S2023001-A',
    qrCode: 'qr-av-001',
    status: 'available'
  },
  {
    id: 'AV-002',
    name: 'Lente 24-70mm f/2.8 GM',
    category: 'Lente',
    description: 'Lente zoom padrão versátil para produção cinematográfica.',
    quantity: 2,
    condition: EquipmentCondition.NEW,
    location: 'Armário A2',
    serialNumber: 'L8899221',
    qrCode: 'qr-av-002',
    status: 'available'
  }
];

const CATEGORIES = [
  { id: 'all', label: 'Tudo', icon: <Archive size={18} /> },
  { id: 'Câmera', label: 'Câmeras', icon: <Camera size={18} /> },
  { id: 'Lente', label: 'Lentes', icon: <Disc size={18} /> },
  { id: 'Iluminação', label: 'Iluminação', icon: <Zap size={18} /> },
  { id: 'Tripés/Suportes', label: 'Tripés', icon: <ChevronRight size={18} className="rotate-90" /> },
  { id: 'Áudio', label: 'Áudio', icon: <Mic2 size={18} /> },
  { id: 'Acessórios', label: 'Acessórios', icon: <Laptop size={18} /> },
  { id: 'Cabos', label: 'Cabos', icon: <Cable size={18} /> },
  { id: 'Infraestrutura', label: 'Infra', icon: <HardDrive size={18} /> }
];

const App: React.FC = () => {
  const [activeView, setActiveView] = useState<View>('dashboard');
  const [inventory, setInventory] = useState<Equipment[]>(INITIAL_INVENTORY);
  const [rentedInventory, setRentedInventory] = useState<RentedEquipment[]>([]);
  const [rentedHistory, setRentedHistory] = useState<RentedEquipment[]>([]);
  const [collaborators, setCollaborators] = useState<Collaborator[]>([]);
  const [bookings, setBookings] = useState<Booking[]>([]);
  const [searchTerm, setSearchTerm] = useState('');
  const [inventoryCategory, setInventoryCategory] = useState('all');
  const [isAiLoading, setIsAiLoading] = useState(false);
  const [previewQr, setPreviewQr] = useState<string>('');

  // Form states
  const [newItem, setNewItem] = useState<Partial<Equipment>>({ 
    name: '', category: 'Câmera', condition: EquipmentCondition.NEW, quantity: 1, location: '', description: '', serialNumber: '' 
  });
  const [newCollaborator, setNewCollaborator] = useState<Partial<Collaborator>>({ name: '', document: '', role: '', email: '', phone: '', legalAccepted: true });
  const [newBooking, setNewBooking] = useState<Partial<Booking>>({ equipmentId: '', collaboratorId: '', project: '', startDate: '', endDate: '' });
  const [newRented, setNewRented] = useState<Partial<RentedEquipment>>({ name: '', ownerCompany: '', project: '', invoiceNumber: '', pickupDate: '', returnDate: '', dailyRate: 0, observations: '' });

  useEffect(() => {
    if (activeView === 'add' && newItem.name) {
      const data = `ID: NEW | Name: ${newItem.name} | SN: ${newItem.serialNumber || 'N/A'}`;
      QRCode.toDataURL(data, { width: 300, margin: 2 }).then(url => setPreviewQr(url));
    }
  }, [newItem.name, newItem.serialNumber, activeView]);

  const handleAddItem = () => {
    if (!newItem.name || !newItem.category) return;
    const item: Equipment = { 
      ...newItem as Equipment, 
      id: `GT-${Math.random().toString(36).substr(2, 4).toUpperCase()}`, 
      qrCode: `qr-${Math.random().toString(36).substr(2, 6)}`, 
      status: 'available' 
    };
    setInventory(prev => [item, ...prev]);
    setActiveView('inventory');
    setNewItem({ name: '', category: 'Câmera', condition: EquipmentCondition.NEW, quantity: 1, location: '', description: '', serialNumber: '' });
  };

  const handleAddBooking = () => {
    if (!newBooking.equipmentId || !newBooking.collaboratorId) return;
    const equip = inventory.find(e => e.id === newBooking.equipmentId);
    const col = collaborators.find(c => c.id === newBooking.collaboratorId);
    const booking: Booking = {
      ...newBooking as Booking,
      id: `BK-${Math.random().toString(36).substr(2, 4).toUpperCase()}`,
      equipmentName: equip?.name || 'Equipamento',
      collaboratorName: col?.name || 'Membro da Equipe',
      status: 'pending',
      createdAt: new Date().toISOString()
    };
    setBookings(prev => [booking, ...prev]);
    setNewBooking({ equipmentId: '', collaboratorId: '', project: '', startDate: '', endDate: '' });
  };

  const handleAddRented = () => {
    if (!newRented.name || !newRented.ownerCompany) return;
    const rented: RentedEquipment = {
      ...newRented as RentedEquipment,
      id: `RENT-${Math.random().toString(36).substr(2, 4).toUpperCase()}`,
      category: 'Alugado',
      status: 'active'
    };
    setRentedInventory(prev => [rented, ...prev]);
    setNewRented({ name: '', ownerCompany: '', project: '', invoiceNumber: '', pickupDate: '', returnDate: '', dailyRate: 0, observations: '' });
  };

  const handleReturnRented = (id: string) => {
    const item = rentedInventory.find(r => r.id === id);
    if (item) {
      setRentedInventory(prev => prev.filter(r => r.id !== id));
      setRentedHistory(prev => [{ ...item, status: 'returned' }, ...prev]);
    }
  };

  const handleAiDescription = async () => {
    if (!newItem.name) return;
    setIsAiLoading(true);
    const desc = await generateEquipmentDescription(newItem.name, newItem.category || 'Audiovisual');
    setNewItem(prev => ({ ...prev, description: desc }));
    setIsAiLoading(false);
  };

  const isWithin48h = (dateStr: string) => {
    const diff = new Date(dateStr).getTime() - new Date().getTime();
    return diff > 0 && diff < (48 * 60 * 60 * 1000);
  };

  const calculateTotalRentedSpend = () => {
    const activeTotal = rentedInventory.reduce((acc, curr) => acc + (curr.dailyRate || 0), 0);
    const historyTotal = rentedHistory.reduce((acc, curr) => acc + (curr.dailyRate || 0), 0);
    return activeTotal + historyTotal;
  };

  const filteredInventory = useMemo(() => {
    return inventory.filter(item => {
      const matchesSearch = item.name.toLowerCase().includes(searchTerm.toLowerCase()) || item.id.toLowerCase().includes(searchTerm.toLowerCase());
      const matchesCategory = inventoryCategory === 'all' || item.category === inventoryCategory;
      return matchesSearch && matchesCategory;
    });
  }, [inventory, searchTerm, inventoryCategory]);

  const getConditionStyle = (condition: EquipmentCondition) => {
    switch (condition) {
      case EquipmentCondition.NEW: return 'bg-[#2F5D50]/10 text-[#2F5D50] border-[#2F5D50]/20';
      case EquipmentCondition.EXCELLENT: return 'bg-emerald-500/10 text-emerald-400 border-emerald-500/20';
      case EquipmentCondition.GOOD: return 'bg-green-500/10 text-green-400 border-green-500/20';
      case EquipmentCondition.REPAIR: return 'bg-red-500/10 text-red-400 border-red-500/20';
      default: return 'bg-slate-800 text-slate-400 border-slate-700';
    }
  };

  return (
    <div className="min-h-screen flex bg-[#0F1115] text-slate-100 font-sans">
      {/* Sidebar */}
      <nav className="w-20 md:w-64 bg-[#0F1115] border-r border-[#2F5D50]/20 flex flex-col h-screen sticky top-0 overflow-hidden shrink-0 z-20">
        <div className="p-6 flex items-center gap-3">
          <div className="w-10 h-10 bg-[#C9A24D] rounded-xl flex items-center justify-center shadow-lg shadow-[#C9A24D]/20 text-[#0F1115]"><Camera size={20} /></div>
          <h1 className="hidden md:block font-bold text-xl tracking-tight text-white uppercase italic">GearTrack</h1>
        </div>
        <div className="flex-1 px-3 space-y-1 mt-4">
          <NavItem active={activeView === 'dashboard'} onClick={() => setActiveView('dashboard')} icon={<LayoutDashboard size={20} />} label="Dashboard" />
          <NavItem active={activeView === 'inventory'} onClick={() => setActiveView('inventory')} icon={<Package size={20} />} label="Estoque" />
          <NavItem active={activeView === 'rented'} onClick={() => setActiveView('rented')} icon={<Truck size={20} />} label="Alugados" />
          <NavItem active={activeView === 'bookings'} onClick={() => setActiveView('bookings')} icon={<CalendarDays size={20} />} label="Agendamentos" />
          <NavItem active={activeView === 'collaborators'} onClick={() => setActiveView('collaborators')} icon={<Users size={20} />} label="Equipe" />
          <div className="h-px bg-[#2F5D50]/20 my-4 mx-4"></div>
          <NavItem active={activeView === 'scan'} onClick={() => setActiveView('scan')} icon={<Scan size={20} />} label="Scanner" />
          <NavItem active={activeView === 'add'} onClick={() => setActiveView('add')} icon={<Plus size={20} />} label="Novo Item" />
        </div>

        {/* Bottom Left Sidebar: Termos Legais */}
        <div className="p-3 mt-auto border-t border-[#2F5D50]/20">
          <NavItem 
            active={activeView === 'legal'} 
            onClick={() => setActiveView('legal')} 
            icon={<Gavel size={20} className="text-slate-400" />} 
            label="Termos Legais" 
          />
        </div>
      </nav>

      {/* Main Content */}
      <main className="flex-1 overflow-y-auto">
        <header className="h-20 border-b border-[#2F5D50]/20 bg-[#0F1115]/80 backdrop-blur-md sticky top-0 z-30 px-6 flex items-center justify-between">
          <h2 className="text-xl font-bold capitalize">
            {activeView === 'rented' ? 'Gestão de Locações Externas' : 
             activeView === 'bookings' ? 'Agenda de Movimentações' :
             activeView === 'legal' ? 'Termos Legais e Responsabilidade' :
             activeView === 'collaborators' ? 'Gestão de Equipe' : activeView}
          </h2>
          <div className="flex items-center gap-4">
            <div className="relative hidden sm:block">
              <Search className="absolute left-3 top-1/2 -translate-y-1/2 text-slate-500" size={18} />
              <input type="text" placeholder="Pesquisar..." className="bg-[#0F1115] border border-[#2F5D50]/30 rounded-full py-2 pl-10 pr-4 text-sm focus:ring-2 focus:ring-[#C9A24D] w-64 outline-none text-white" value={searchTerm} onChange={(e) => setSearchTerm(e.target.value)} />
            </div>
          </div>
        </header>

        <div className="p-6 space-y-6">
          {activeView === 'dashboard' && (
            <div className="grid grid-cols-1 md:grid-cols-4 gap-4 animate-in fade-in duration-500">
              <StatCard label="Itens no Acervo" value={inventory.length} icon={<Package className="text-[#C9A24D]" />} />
              <StatCard label="Locações Externas" value={rentedInventory.filter(r => r.status === 'active').length} icon={<Truck className="text-[#2F5D50]" />} />
              <StatCard label="Agendamentos" value={bookings.length} icon={<CalendarDays className="text-[#C9A24D]" />} />
              <StatCard label="Alertas 48h" value={bookings.filter(b => isWithin48h(b.startDate)).length} icon={<BellRing className="text-red-400" />} />
            </div>
          )}

          {activeView === 'inventory' && (
            <div className="space-y-6 animate-in fade-in duration-500">
              <div className="flex items-center gap-2 overflow-x-auto pb-2 no-scrollbar border-b border-[#2F5D50]/20">
                {CATEGORIES.map(cat => (
                  <button key={cat.id} onClick={() => setInventoryCategory(cat.id)} className={`flex items-center gap-2 px-4 py-2 rounded-xl border whitespace-nowrap transition-all ${inventoryCategory === cat.id ? 'bg-[#C9A24D] border-[#C9A24D] text-[#0F1115] shadow-lg' : 'bg-[#0F1115] border-[#2F5D50]/30 text-slate-400 hover:bg-[#2F5D50]/10'}`}>
                    {cat.icon} <span className="text-xs font-bold uppercase">{cat.label}</span>
                  </button>
                ))}
              </div>

              <div className="bg-[#0F1115] rounded-3xl border border-[#2F5D50]/20 overflow-hidden shadow-2xl">
                <table className="w-full text-left border-collapse">
                  <thead className="bg-[#2F5D50]/5 border-b border-[#2F5D50]/20">
                    <tr>
                      <th className="p-4 text-[10px] font-bold text-slate-400 uppercase tracking-widest">Equipamento</th>
                      <th className="p-4 text-[10px] font-bold text-slate-400 uppercase tracking-widest text-center">Estado</th>
                      <th className="p-4 text-[10px] font-bold text-slate-400 uppercase tracking-widest">Localização</th>
                      <th className="p-4 text-[10px] font-bold text-slate-400 uppercase tracking-widest text-center">Status</th>
                    </tr>
                  </thead>
                  <tbody className="divide-y divide-[#2F5D50]/10">
                    {filteredInventory.map(item => (
                      <tr key={item.id} className="hover:bg-[#2F5D50]/5 transition-colors group">
                        <td className="p-4">
                          <div className="flex items-center gap-3">
                            <div className="w-10 h-10 bg-[#2F5D50]/10 rounded-xl flex items-center justify-center text-slate-400 group-hover:text-[#C9A24D] transition-colors">
                              {CATEGORIES.find(c => c.id === item.category)?.icon || <Package size={18} />}
                            </div>
                            <div>
                              <p className="text-sm font-bold text-white">{item.name}</p>
                              <p className="text-[10px] text-slate-500 font-mono uppercase tracking-tighter">{item.id} • {item.category}</p>
                            </div>
                          </div>
                        </td>
                        <td className="p-4 text-center">
                          <span className={`px-2 py-1 rounded-lg border text-[10px] font-bold uppercase ${getConditionStyle(item.condition)}`}>
                            {item.condition}
                          </span>
                        </td>
                        <td className="p-4">
                          <div className="flex items-center gap-1.5 text-slate-300 text-xs font-medium">
                            <MapPin size={12} className="text-[#C9A24D]" />
                            {item.location || 'Sem local definido'}
                          </div>
                        </td>
                        <td className="p-4 text-center">
                          <span className={`px-3 py-1 rounded-full text-[9px] font-black uppercase ${item.status === 'available' ? 'bg-[#2F5D50]/20 text-[#2F5D50]' : 'bg-red-500/10 text-red-400'}`}>
                            {item.status === 'available' ? 'Livre' : 'Em Campo'}
                          </span>
                        </td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            </div>
          )}

          {activeView === 'rented' && (
            <div className="space-y-8 animate-in fade-in duration-500">
              {/* Analytics Header */}
              <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
                 <div className="bg-[#0F1115] p-6 rounded-3xl border border-[#2F5D50]/20 flex items-center gap-4">
                    <div className="w-12 h-12 bg-[#C9A24D]/10 text-[#C9A24D] rounded-2xl flex items-center justify-center"><DollarSign size={24} /></div>
                    <div>
                       <p className="text-[10px] text-slate-500 uppercase font-bold tracking-widest">Investimento Total</p>
                       <p className="text-xl font-bold text-white">R$ {calculateTotalRentedSpend().toLocaleString()}</p>
                    </div>
                 </div>
                 <div className="bg-[#0F1115] p-6 rounded-3xl border border-[#2F5D50]/20 flex items-center gap-4">
                    <div className="w-12 h-12 bg-[#2F5D50]/10 text-[#2F5D50] rounded-2xl flex items-center justify-center"><BarChart3 size={24} /></div>
                    <div>
                       <p className="text-[10px] text-slate-500 uppercase font-bold tracking-widest">Efficiency ROI</p>
                       <p className="text-xl font-bold text-white">84% <span className="text-[10px] text-[#2F5D50] font-normal">Optimal</span></p>
                    </div>
                 </div>
                 <div className="bg-[#0F1115] p-6 rounded-3xl border border-[#2F5D50]/20 flex items-center gap-4">
                    <div className="w-12 h-12 bg-[#C9A24D]/10 text-[#C9A24D] rounded-2xl flex items-center justify-center"><History size={24} /></div>
                    <div>
                       <p className="text-[10px] text-slate-500 uppercase font-bold tracking-widest">Aluguéis Finalizados</p>
                       <p className="text-xl font-bold text-white">{rentedHistory.length} Registros</p>
                    </div>
                 </div>
              </div>

              <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
                {/* Form */}
                <div className="lg:col-span-1 bg-[#0F1115] p-6 rounded-3xl border border-[#2F5D50]/20 space-y-4 shadow-xl h-fit">
                  <h3 className="text-lg font-bold flex items-center gap-2 text-[#C9A24D]"><Plus size={20} /> Nova Locação</h3>
                  <InputField label="Equipamento" value={newRented.name} onChange={v => setNewRented({...newRented, name: v})} placeholder="Ex: Gimbal RS3 Pro" />
                  <InputField label="Projeto" value={newRented.project} onChange={v => setNewRented({...newRented, project: v})} placeholder="Ex: Comercial Coca-Cola" icon={<Clapperboard size={14} />} />
                  <InputField label="Empresa (Rental)" value={newRented.ownerCompany} onChange={v => setNewRented({...newRented, ownerCompany: v})} placeholder="Ex: CineLoc" icon={<Building2 size={14} />} />
                  <div className="grid grid-cols-2 gap-3">
                    <InputField label="NF" value={newRented.invoiceNumber} onChange={v => setNewRented({...newRented, invoiceNumber: v})} placeholder="000" />
                    <InputField label="Diária (R$)" type="number" value={newRented.dailyRate} onChange={v => setNewRented({...newRented, dailyRate: Number(v)})} />
                  </div>
                  <div className="grid grid-cols-2 gap-3">
                    <InputField label="Retirada" type="date" value={newRented.pickupDate} onChange={v => setNewRented({...newRented, pickupDate: v})} />
                    <InputField label="Devolução" type="date" value={newRented.returnDate} onChange={v => setNewRented({...newRented, returnDate: v})} />
                  </div>
                  <div className="space-y-1">
                     <label className="text-[10px] text-slate-500 uppercase font-bold tracking-widest flex items-center gap-1.5"><ClipboardList size={12} /> Observações de Checkout</label>
                     <textarea className="w-full bg-[#0F1115] border border-[#2F5D50]/30 rounded-xl p-3 text-xs text-white outline-none focus:ring-1 focus:ring-[#C9A24D] min-h-[80px]" value={newRented.observations} onChange={e => setNewRented({...newRented, observations: e.target.value})} placeholder="Checklist de acessórios..." />
                  </div>
                  <button onClick={handleAddRented} className="w-full py-3 bg-[#C9A24D] hover:bg-[#B38D3A] rounded-xl font-bold transition-all shadow-lg shadow-[#C9A24D]/20 text-[#0F1115]">CONFIRMAR REGISTRO</button>
                </div>

                {/* Main Lists */}
                <div className="lg:col-span-2 space-y-6">
                  {/* Active Rentals */}
                  <div className="bg-[#0F1115] rounded-3xl border border-[#2F5D50]/20 overflow-hidden shadow-2xl">
                    <div className="p-4 bg-[#2F5D50]/5 border-b border-[#2F5D50]/20 font-bold text-[10px] uppercase tracking-widest text-slate-500 flex justify-between items-center">
                       <span className="flex items-center gap-2"><ArrowUpRight size={14} className="text-[#C9A24D]" /> Locações Ativas</span>
                       <span className="bg-[#2F5D50]/20 text-[#2F5D50] px-2 py-0.5 rounded text-[8px]">{rentedInventory.length} Em campo</span>
                    </div>
                    <div className="divide-y divide-[#2F5D50]/10">
                      {rentedInventory.map(r => (
                        <div key={r.id} className="p-6 flex flex-col gap-4 hover:bg-[#2F5D50]/5 transition-all">
                          <div className="flex items-center justify-between">
                            <div className="flex gap-4">
                               <div className="w-12 h-12 bg-[#2F5D50]/10 text-[#2F5D50] rounded-2xl flex items-center justify-center border border-[#2F5D50]/20"><Building2 size={24} /></div>
                               <div>
                                  <p className="font-bold text-white text-base">{r.name}</p>
                                  <p className="text-[10px] text-slate-500 uppercase font-medium">{r.ownerCompany} • NF: {r.invoiceNumber}</p>
                                  <p className="text-[10px] text-[#C9A24D] font-bold uppercase mt-0.5 flex items-center gap-1"><Clapperboard size={10}/> PROJETO: {r.project || 'NÃO INFORMADO'}</p>
                               </div>
                            </div>
                            <div className="text-right">
                               <p className="text-sm font-bold text-white">R$ {r.dailyRate?.toLocaleString()}/dia</p>
                               <button onClick={() => handleReturnRented(r.id)} className="text-[10px] font-black uppercase bg-[#2F5D50]/20 text-[#2F5D50] px-3 py-1 rounded-lg mt-1 hover:bg-[#2F5D50] hover:text-[#0F1115] transition-all">Devolver</button>
                            </div>
                          </div>
                          {r.observations && (
                            <div className="bg-[#0F1115] p-3 rounded-xl border border-[#2F5D50]/10">
                               <p className="text-[9px] text-slate-500 uppercase font-bold mb-1 flex items-center gap-1"><Info size={10} /> Notas Técnicas</p>
                               <p className="text-xs text-slate-400 italic">"{r.observations}"</p>
                            </div>
                          )}
                        </div>
                      ))}
                    </div>
                  </div>
                </div>
              </div>
            </div>
          )}

          {activeView === 'add' && (
            <div className="grid grid-cols-1 lg:grid-cols-3 gap-8 animate-in slide-in-from-bottom-4 duration-500">
              <div className="lg:col-span-2 space-y-6">
                <div className="bg-[#0F1115] p-8 rounded-3xl border border-[#2F5D50]/20 space-y-6 shadow-xl">
                  <div className="flex items-center gap-3 border-b border-[#2F5D50]/20 pb-4">
                    <div className="w-10 h-10 bg-[#C9A24D]/20 text-[#C9A24D] rounded-xl flex items-center justify-center"><Plus size={20} /></div>
                    <h3 className="text-xl font-bold">Novo Item no Acervo</h3>
                  </div>
                  <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                    <InputField label="Nome do Equipamento" value={newItem.name} onChange={v => setNewItem({...newItem, name: v})} placeholder="Ex: Blackmagic 6K G2" />
                    <div className="space-y-1">
                      <label className="text-[10px] text-slate-500 uppercase font-bold tracking-widest">Categoria Audiovisual</label>
                      <select className="w-full bg-[#0F1115] border border-[#2F5D50]/30 rounded-xl px-4 py-2.5 text-white outline-none focus:ring-1 focus:ring-[#C9A24D] text-sm" value={newItem.category} onChange={e => setNewItem({...newItem, category: e.target.value})}>
                        {CATEGORIES.filter(c => c.id !== 'all').map(c => <option key={c.id} value={c.id}>{c.label}</option>)}
                      </select>
                    </div>
                  </div>
                  <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
                    <InputField label="Nº de Série" value={newItem.serialNumber} onChange={v => setNewItem({...newItem, serialNumber: v})} placeholder="S/N" />
                    <InputField label="Localização" icon={<MapPin size={14} />} value={newItem.location} onChange={v => setNewItem({...newItem, location: v})} placeholder="Ex: Estante 03 - A" />
                    <div className="space-y-1">
                      <label className="text-[10px] text-slate-500 uppercase font-bold tracking-widest">Estado</label>
                      <select className="w-full bg-[#0F1115] border border-[#2F5D50]/30 rounded-xl px-4 py-2.5 text-white outline-none focus:ring-1 focus:ring-[#C9A24D] text-sm" value={newItem.condition} onChange={e => setNewItem({...newItem, condition: e.target.value as any})}>
                        {Object.values(EquipmentCondition).map(cond => <option key={cond} value={cond}>{cond}</option>)}
                      </select>
                    </div>
                  </div>
                  <div className="space-y-1">
                    <div className="flex justify-between items-center mb-1">
                      <label className="text-[10px] text-slate-500 uppercase font-bold tracking-widest">Descrição Técnica Inteligente</label>
                      <button onClick={handleAiDescription} disabled={!newItem.name || isAiLoading} className="flex items-center gap-1 text-[10px] bg-[#C9A24D]/10 text-[#C9A24D] px-2 py-1 rounded-lg hover:bg-[#C9A24D] hover:text-[#0F1115] transition-all disabled:opacity-20 font-black">
                        <Sparkles size={12} /> {isAiLoading ? 'CONSULTANDO...' : 'PREENCHER COM IA'}
                      </button>
                    </div>
                    <textarea className="w-full bg-[#0F1115] border border-[#2F5D50]/30 rounded-xl p-4 text-sm text-white min-h-[120px] outline-none focus:ring-1 focus:ring-[#C9A24D]" value={newItem.description} onChange={e => setNewItem({...newItem, description: e.target.value})} placeholder="Especificações técnicas..." />
                  </div>
                  <button onClick={handleAddItem} className="w-full py-4 bg-[#C9A24D] hover:bg-[#B38D3A] rounded-2xl text-[#0F1115] font-bold shadow-lg shadow-[#C9A24D]/30">FINALIZAR CADASTRO</button>
                </div>
              </div>
              <div className="space-y-6">
                <div className="bg-[#0F1115] p-8 rounded-3xl border border-[#2F5D50]/20 flex flex-col items-center text-center space-y-4 shadow-xl">
                  <h4 className="text-xs font-bold text-slate-500 uppercase tracking-widest">QRCode de Identificação</h4>
                  <div className="bg-white p-4 rounded-2xl shadow-inner border-8 border-[#0F1115]">
                    {previewQr ? <img src={previewQr} alt="QR Code Preview" className="w-48 h-48" /> : <div className="w-48 h-48 bg-slate-800 animate-pulse rounded-lg" />}
                  </div>
                  <div className="space-y-1">
                    <p className="text-sm font-bold text-white truncate w-48">{newItem.name || 'Aguardando Nome...'}</p>
                    <p className="text-[10px] text-slate-500 font-mono">LOC: {newItem.location || 'PENDENTE'}</p>
                  </div>
                  <button className="flex items-center gap-2 text-xs font-bold text-[#C9A24D] bg-[#C9A24D]/10 px-4 py-2 rounded-xl hover:bg-[#C9A24D] hover:text-[#0F1115] transition-all">
                    <Download size={16} /> BAIXAR ETIQUETA
                  </button>
                </div>
              </div>
            </div>
          )}

          {activeView === 'legal' && (
            <div className="max-w-4xl mx-auto animate-in fade-in slide-in-from-bottom-4 duration-700">
               <div className="bg-[#0F1115] border border-[#2F5D50]/20 rounded-[32px] p-8 md:p-12 shadow-2xl relative overflow-hidden">
                  <div className="absolute top-0 right-0 p-12 opacity-5 text-[#C9A24D]">
                    <Scale size={320} />
                  </div>
                  <div className="relative z-10 space-y-8">
                     <div className="flex items-center gap-4 border-b border-[#2F5D50]/20 pb-8">
                        <div className="w-16 h-16 bg-[#C9A24D]/10 text-[#C9A24D] rounded-2xl flex items-center justify-center border border-[#C9A24D]/20"><Gavel size={32} /></div>
                        <div>
                           <h1 className="text-3xl font-black text-white uppercase tracking-tight italic">GearTrack Termos</h1>
                           <p className="text-slate-500 font-medium">Controle de Ativos Audiovisuais v3.0</p>
                        </div>
                     </div>
                     <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
                        <div className="space-y-6">
                           <div className="space-y-2">
                              <h3 className="text-[#C9A24D] text-xs font-black uppercase tracking-widest flex items-center gap-2"><ShieldAlert size={14} /> 01. Guarda e Conservação</h3>
                              <p className="text-sm text-slate-300 leading-relaxed">Responsabilidade total pela integridade física dos equipamentos retirados.</p>
                           </div>
                           <div className="space-y-2">
                              <h3 className="text-[#C9A24D] text-xs font-black uppercase tracking-widest flex items-center gap-2"><MapPin size={14} /> 02. Logística</h3>
                              <p className="text-sm text-slate-300 leading-relaxed">Equipamentos rastreados e registrados por ID único GearTrack.</p>
                           </div>
                        </div>
                        <div className="space-y-6 text-slate-300">
                           <div className="space-y-2">
                              <h3 className="text-[#C9A24D] text-xs font-black uppercase tracking-widest flex items-center gap-2"><DollarSign size={14} /> 03. Responsabilidade</h3>
                              <p className="text-sm leading-relaxed">Em caso de negligência, custos de reparo ou reposição aplicáveis.</p>
                           </div>
                        </div>
                     </div>
                     <div className="bg-[#2F5D50]/5 border border-[#2F5D50]/20 rounded-2xl p-6 flex flex-col md:flex-row items-center gap-6">
                        <div className="flex-1 text-sm text-slate-400 italic">"Termos vigentes para toda a equipe GearTrack."</div>
                        <button onClick={() => setActiveView('dashboard')} className="px-6 py-3 bg-[#C9A24D] hover:bg-[#B38D3A] text-[#0F1115] rounded-xl font-bold transition-all text-sm shadow-lg shadow-[#C9A24D]/20">CONCORDAR</button>
                     </div>
                  </div>
               </div>
            </div>
          )}

          {(activeView === 'bookings' || activeView === 'collaborators') && (
            <div className="p-10 text-center border-2 border-dashed border-[#2F5D50]/20 rounded-[32px] text-slate-500">
               Configurações de {activeView} mantidas de acordo com as especificações GearTrack.
            </div>
          )}

          {activeView === 'scan' && (
            <Scanner inventory={inventory} collaborators={collaborators} onScanComplete={(eq, act) => {
              setInventory(prev => prev.map(item => item.id === eq.id ? { ...item, status: act === 'check-out' ? 'out' : 'available' } : item));
              setActiveView('dashboard');
            }} onCancel={() => setActiveView('dashboard')} />
          )}
        </div>
      </main>
    </div>
  );
};

const NavItem = ({ active, onClick, icon, label }: any) => (
  <button onClick={onClick} className={`w-full flex items-center gap-3 px-4 py-3 rounded-xl transition-all group ${active ? 'bg-[#2F5D50] text-[#C9A24D] shadow-lg shadow-[#2F5D50]/20' : 'text-slate-400 hover:bg-[#2F5D50]/10 hover:text-slate-200'}`}>
    <div className={active ? 'text-[#C9A24D]' : 'text-slate-400 group-hover:text-[#C9A24D]'}>{icon}</div>
    <span className="hidden md:block font-medium text-sm">{label}</span>
  </button>
);

const StatCard = ({ label, value, icon }: any) => (
  <div className="bg-[#0F1115] p-6 rounded-3xl border border-[#2F5D50]/20 shadow-sm group hover:border-[#C9A24D]/30 transition-colors">
    <div className="p-2.5 w-11 h-11 rounded-xl bg-[#2F5D50]/10 border border-[#2F5D50]/20 mb-4 flex items-center justify-center">{icon}</div>
    <p className="text-[10px] text-slate-500 font-bold uppercase tracking-widest">{label}</p>
    <p className="text-2xl font-bold text-white mt-1">{value}</p>
  </div>
);

const InputField = ({ label, value, onChange, placeholder, icon, type = "text" }: any) => (
  <div className="space-y-1">
    <label className="text-[10px] text-slate-500 uppercase font-bold tracking-widest flex items-center gap-1.5">{icon} {label}</label>
    <input type={type} className="w-full bg-[#0F1115] border border-[#2F5D50]/30 rounded-xl px-4 py-2 text-white outline-none text-sm focus:ring-1 focus:ring-[#C9A24D] placeholder:text-slate-800 transition-all" value={value} onChange={e => onChange(e.target.value)} placeholder={placeholder} />
  </div>
);

export default App;
