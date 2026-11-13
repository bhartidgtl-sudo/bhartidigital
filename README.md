import React, { useState } from 'react';
import { 
  CreditCard, 
  IndianRupee, 
  AlertCircle, 
  Send, 
  Download, 
  Printer, 
  CheckCircle2, 
  Search, 
  FileText, 
  ArrowUpRight,
  Sparkles,
  MessageSquare,
  Lock,
  Calendar,
  QrCode,
  Upload,
  Trash2,
  Smartphone
} from 'lucide-react';
import { Student, Invoice, LibrarySettings, formatStudentShifts } from '../types';
import { generateWhatsAppReminderUrl, sounds } from '../utils/notifications';
import { YearlyFeeMatrix } from './YearlyFeeMatrix';
import { PhonePeScannerCard, getEffectiveUpiDetails } from './PhonePeScannerCard';
import confetti from 'canvas-confetti';

interface BillingViewProps {
  students: Student[];
  invoices: Invoice[];
  settings: LibrarySettings;
  isAdmin: boolean;
  onRequireAdmin: () => void;
  onUpdateSettings?: (updates: Partial<LibrarySettings>) => void;
  onRecordPayment: (studentId: string, amount: number, method: 'UPI' | 'CASH' | 'CARD', targetYearMonth?: string) => Invoice;
  onOpenNotifications: () => void;
}

export const BillingView: React.FC<BillingViewProps> = ({
  students,
  invoices,
  settings,
  isAdmin,
  onRequireAdmin,
  onUpdateSettings,
  onRecordPayment,
  onOpenNotifications,
}) => {
  const [billingSubTab, setBillingSubTab] = useState<'LEDGER' | 'YEARLY_MATRIX' | 'PHONEPE_SCANNER'>('LEDGER');
  const upiDefaults = getEffectiveUpiDetails(settings);
  const [upiNumberInput, setUpiNumberInput] = useState(upiDefaults.upiNumber);
  const [upiIdInput, setUpiIdInput] = useState(upiDefaults.upiId);
  const [upiPayeeInput, setUpiPayeeInput] = useState(upiDefaults.payeeName);
  const [customQrInput, setCustomQrInput] = useState<string | undefined>(settings.phonePeQrImageUrl);
  const [upiSavedToast, setUpiSavedToast] = useState(false);
  const [selectedStudentForPay, setSelectedStudentForPay] = useState<Student | null>(null);
  const [targetYearMonthForPay, setTargetYearMonthForPay] = useState<string | undefined>(undefined);
  const [isStudentPickerOpen, setIsStudentPickerOpen] = useState(false);
  const [studentPickerQuery, setStudentPickerQuery] = useState('');
  const [paymentAmount, setPaymentAmount] = useState<number>(500);
  const [paymentMethod, setPaymentMethod] = useState<'UPI' | 'CASH' | 'CARD'>('UPI');
  const [receiptToPrint, setReceiptToPrint] = useState<Invoice | null>(null);
  const [searchInvoiceQuery, setSearchInvoiceQuery] = useState('');

  const overdueStudents = students.filter(s => s.paymentStatus === 'OVERDUE');
  const dueSoonStudents = students.filter(s => s.paymentStatus === 'DUE_SOON');

  const currentYearMonth = new Date().toISOString().slice(0, 7);
  const currentMonthInvoices = invoices.filter(
    inv => inv.status === 'PAID' && (inv.periodStart ? inv.periodStart.startsWith(currentYearMonth) : inv.paidDate.startsWith(currentYearMonth))
  );
  const totalCollected = currentMonthInvoices.reduce((acc, inv) => acc + inv.amount, 0);
  const totalPending = [...overdueStudents, ...dueSoonStudents].reduce((acc, s) => acc + s.monthlyFee, 0);

  const handleOpenPay = (student: Student, yearMonth?: string) => {
    setSelectedStudentForPay(student);
    setTargetYearMonthForPay(yearMonth);
    setPaymentAmount(student.monthlyFee);
    setIsStudentPickerOpen(false);
  };

  const handleConfirmPayment = (e: React.FormEvent) => {
    e.preventDefault();
    if (!selectedStudentForPay) return;

    const newInvoice = onRecordPayment(selectedStudentForPay.id, paymentAmount, paymentMethod, targetYearMonthForPay);
    sounds.playSuccess();
    try {
      confetti({
        particleCount: 70,
        spread: 80,
        origin: { y: 0.6 },
      });
    } catch {
      // ignore
    }

    setSelectedStudentForPay(null);
    setReceiptToPrint(newInvoice);
  };

  const filteredInvoices = [...invoices]
    .sort((a, b) => b.paidDate.localeCompare(a.paidDate))
    .filter(inv => 
      inv.studentName.toLowerCase().includes(searchInvoiceQuery.toLowerCase()) ||
      inv.id.toLowerCase().includes(searchInvoiceQuery.toLowerCase()) ||
      inv.transactionRef.toLowerCase().includes(searchInvoiceQuery.toLowerCase())
    );

  return (
    <div className="space-y-6 pb-20 md:pb-8">
      {/* Top Banner */}
      <div className="flex flex-col sm:flex-row sm:items-center sm:justify-between gap-4">
        <div>
          <h2 className="text-xl font-bold tracking-tight text-slate-900 dark:text-white">
            Billing & Revenue Management
          </h2>
          <p className="text-xs text-slate-500">
            Automated monthly fee renewal, digital receipt generation, and reminder dispatch
          </p>
        </div>

        <div className="flex flex-wrap items-center gap-2">
          <button
            onClick={() => {
              if (!isAdmin) {
                onRequireAdmin();
                return;
              }
              setStudentPickerQuery('');
              setIsStudentPickerOpen(true);
            }}
            className="flex items-center gap-1.5 px-4 py-2 text-xs font-bold text-white bg-emerald-600 hover:bg-emerald-700 rounded-xl shadow-sm transition-all min-h-[40px]"
          >
            {isAdmin ? <IndianRupee className="w-4 h-4" /> : <Lock className="w-3.5 h-3.5" />}
            <span>{isAdmin ? '+ Collect Student Fee' : 'Admin Fee Collection'}</span>
          </button>
          <button
            onClick={onOpenNotifications}
            className="flex items-center gap-1.5 px-4 py-2 text-xs font-bold text-white bg-indigo-600 hover:bg-indigo-700 rounded-xl shadow-sm transition-all min-h-[40px]"
          >
            <Send className="w-4 h-4" />
            <span>Send Payment Reminders ({overdueStudents.length + dueSoonStudents.length})</span>
          </button>
        </div>
      </div>

      {/* Sub-navigation: Fee Ledger vs Jan-Dec Unpaid Tracker vs PhonePe Scanner */}
      <div className="flex flex-wrap items-center gap-1.5 p-1.5 bg-slate-100 dark:bg-slate-800/80 rounded-2xl w-fit">
        <button
          onClick={() => setBillingSubTab('LEDGER')}
          className={`flex items-center gap-2 px-4 py-2 rounded-xl text-xs font-semibold transition-all whitespace-nowrap ${
            billingSubTab === 'LEDGER'
              ? 'bg-white dark:bg-slate-900 text-slate-900 dark:text-white shadow-sm'
              : 'text-slate-600 dark:text-slate-400 hover:text-slate-900'
          }`}
        >
          <CreditCard className="w-4 h-4 text-emerald-600" />
          <span>Dues & Invoices</span>
        </button>
        <button
          onClick={() => setBillingSubTab('YEARLY_MATRIX')}
          className={`flex items-center gap-2 px-4 py-2 rounded-xl text-xs font-semibold transition-all whitespace-nowrap ${
            billingSubTab === 'YEARLY_MATRIX'
              ? 'bg-white dark:bg-slate-900 text-slate-900 dark:text-white shadow-sm'
              : 'text-slate-600 dark:text-slate-400 hover:text-slate-900'
          }`}
        >
          <Calendar className="w-4 h-4 text-rose-600" />
          <span>Jan–Dec Unpaid Tracker</span>
        </button>
        <button
          onClick={() => setBillingSubTab('PHONEPE_SCANNER')}
          className={`flex items-center gap-2 px-4 py-2 rounded-xl text-xs font-semibold transition-all whitespace-nowrap ${
            billingSubTab === 'PHONEPE_SCANNER'
              ? 'bg-white dark:bg-slate-900 text-slate-900 dark:text-white shadow-sm'
              : 'text-slate-600 dark:text-slate-400 hover:text-slate-900'
          }`}
        >
          <QrCode className="w-4 h-4 text-[#5f259f]" />
          <span>UPI & PhonePe Scanner</span>
        </button>
      </div>

      {billingSubTab === 'PHONEPE_SCANNER' ? (
        <div className="grid grid-cols-1 lg:grid-cols-12 gap-6 items-start">
          {/* Left: Official Counter PhonePe Scanner */}
          <div className="lg:col-span-5 space-y-3">
            <PhonePeScannerCard settings={settings} />
            <button
              type="button"
              onClick={() => window.print()}
              className="w-full py-2.5 rounded-xl border border-slate-200 dark:border-slate-700 bg-white dark:bg-slate-900 hover:bg-slate-50 dark:hover:bg-slate-800 text-xs font-semibold text-slate-700 dark:text-slate-200 flex items-center justify-center gap-1.5 transition-colors"
            >
              <Printer className="w-4 h-4" />
              <span>Print PhonePe Counter Standee</span>
            </button>
          </div>

          {/* Right: Admin Configuration for UPI Number, UPI ID & Custom PhonePe QR Upload */}
          <div className="lg:col-span-7 p-5 sm:p-6 rounded-2xl bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-800 space-y-4">
            <div className="flex items-center justify-between border-b border-slate-100 dark:border-slate-800 pb-3">
              <div>
                <h3 className="text-sm font-bold text-slate-900 dark:text-white flex items-center gap-2">
                  <Smartphone className="w-4 h-4 text-[#5f259f]" />
                  <span>Library UPI Number & PhonePe Scanner Setup</span>
                </h3>
                <p className="text-xs text-slate-500 dark:text-slate-400 mt-0.5">
                  Students see this UPI number and PhonePe QR scanner when paying fees online or at the desk.
                </p>
              </div>
              {!isAdmin && (
                <button
                  type="button"
                  onClick={onRequireAdmin}
                  className="px-3 py-1.5 rounded-xl bg-amber-50 dark:bg-amber-950/50 border border-amber-200 dark:border-amber-800 text-amber-800 dark:text-amber-300 text-xs font-semibold flex items-center gap-1.5 shrink-0"
                >
                  <Lock className="w-3.5 h-3.5" />
                  <span>Unlock Admin</span>
                </button>
              )}
            </div>

            <form
              onSubmit={e => {
                e.preventDefault();
                if (!isAdmin) {
                  onRequireAdmin();
                  return;
                }
                onUpdateSettings?.({
                  upiNumber: upiNumberInput.trim(),
                  upiId: upiIdInput.trim(),
                  upiPayeeName: upiPayeeInput.trim() || settings.name,
                  phonePeQrImageUrl: customQrInput,
                });
                setUpiSavedToast(true);
                setTimeout(() => setUpiSavedToast(false), 2500);
              }}
              className="space-y-4"
            >
              <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                <div>
                  <label className="text-xs font-semibold text-slate-700 dark:text-slate-300 block mb-1">
                    PhonePe / UPI Mobile Number *
                  </label>
                  <input
                    type="tel"
                    required
                    disabled={!isAdmin}
                    value={upiNumberInput}
                    onChange={e => setUpiNumberInput(e.target.value)}
                    placeholder="+91 98765 43210"
                    className="w-full px-3.5 py-2.5 rounded-xl border border-slate-200 dark:border-slate-700 bg-white dark:bg-slate-800 text-sm font-mono tabular-nums text-slate-900 dark:text-white disabled:opacity-60 focus:outline-none focus:ring-2 focus:ring-[#5f259f]"
                  />
                </div>

                <div>
                  <label className="text-xs font-semibold text-slate-700 dark:text-slate-300 block mb-1">
                    UPI ID (VPA) *
                  </label>
                  <input
                    type="text"
                    required
                    disabled={!isAdmin}
                    value={upiIdInput}
                    onChange={e => setUpiIdInput(e.target.value)}
                    placeholder="9876543210@ybl"
                    className="w-full px-3.5 py-2.5 rounded-xl border border-slate-200 dark:border-slate-700 bg-white dark:bg-slate-800 text-sm font-mono text-slate-900 dark:text-white disabled:opacity-60 focus:outline-none focus:ring-2 focus:ring-[#5f259f]"
                  />
                </div>
              </div>

              <div>
                <label className="text-xs font-semibold text-slate-700 dark:text-slate-300 block mb-1">
                  Payee / Business Name on Scanner
                </label>
                <input
                  type="text"
                  disabled={!isAdmin}
                  value={upiPayeeInput}
                  onChange={e => setUpiPayeeInput(e.target.value)}
                  placeholder="Bharti Digital Library"
                  className="w-full px-3.5 py-2.5 rounded-xl border border-slate-200 dark:border-slate-700 bg-white dark:bg-slate-800 text-sm text-slate-900 dark:text-white disabled:opacity-60 focus:outline-none focus:ring-2 focus:ring-[#5f259f]"
                />
              </div>

              <div className="p-4 rounded-xl bg-slate-50 dark:bg-slate-800/60 border border-slate-200 dark:border-slate-700 space-y-2.5">
                <div className="flex items-center justify-between gap-2">
                  <div>
                    <span className="text-xs font-bold text-slate-900 dark:text-white block">
                      Custom PhonePe QR Image (Optional)
                    </span>
                    <span className="text-[11px] text-slate-500 dark:text-slate-400">
                      Upload your PhonePe QR screenshot, or leave empty to auto-generate a scannable UPI QR from your UPI ID.
                    </span>
                  </div>
                  {customQrInput && isAdmin && (
                    <button
                      type="button"
                      onClick={() => setCustomQrInput(undefined)}
                      className="px-2.5 py-1.5 rounded-lg border border-rose-200 dark:border-rose-900 text-rose-600 dark:text-rose-400 text-xs font-semibold inline-flex items-center gap-1 shrink-0"
                    >
                      <Trash2 className="w-3.5 h-3.5" />
                      <span>Use Auto QR</span>
                    </button>
                  )}
                </div>

                <label
                  className={`inline-flex items-center gap-2 px-3.5 py-2 rounded-xl border border-slate-200 dark:border-slate-700 bg-white dark:bg-slate-900 text-xs font-semibold text-slate-700 dark:text-slate-200 transition-colors ${
                    isAdmin ? 'cursor-pointer hover:bg-slate-100 dark:hover:bg-slate-800' : 'opacity-60 cursor-not-allowed'
                  }`}
                >
                  <Upload className="w-3.5 h-3.5 text-[#5f259f]" />
                  <span>{customQrInput ? 'Change PhonePe QR Image' : 'Upload PhonePe QR Scanner Photo'}</span>
                  <input
                    type="file"
                    accept="image/*"
                    disabled={!isAdmin}
                    className="hidden"
                    onChange={e => {
                      const file = e.target.files?.[0];
                      if (!file) return;
                      const reader = new FileReader();
                      reader.onload = () => {
                        if (typeof reader.result === 'string') {
                          setCustomQrInput(reader.result);
                        }
                      };
                      reader.readAsDataURL(file);
                    }}
                  />
                </label>
              </div>

              {upiSavedToast && (
                <div className="p-3 rounded-xl bg-emerald-50 dark:bg-emerald-950/40 border border-emerald-200 dark:border-emerald-900/50 text-xs text-emerald-700 dark:text-emerald-300 font-semibold flex items-center gap-2">
                  <CheckCircle2 className="w-4 h-4 shrink-0" />
                  <span>UPI number and PhonePe scanner updated across Billing and Student Portal.</span>
                </div>
              )}

              <div className="flex justify-end">
                <button
                  type="submit"
                  className="px-5 py-2.5 rounded-xl bg-[#5f259f] hover:bg-[#4e1d84] text-white text-xs font-bold flex items-center gap-1.5 transition-colors min-h-[42px]"
                >
                  <CheckCircle2 className="w-4 h-4" />
                  <span>{isAdmin ? 'Save UPI & PhonePe Scanner' : 'Unlock Admin to Save'}</span>
                </button>
              </div>
            </form>
          </div>
        </div>
      ) : billingSubTab === 'YEARLY_MATRIX' ? (
        <YearlyFeeMatrix
          students={students}
          invoices={invoices}
          settings={settings}
          isAdmin={isAdmin}
          onRequireAdmin={onRequireAdmin}
          onCollectMonthFee={(student, yearMonth) => handleOpenPay(student, yearMonth)}
        />
      ) : (
      <>
      {/* KPI Financial Overview Cards */}
      <div className="grid grid-cols-1 sm:grid-cols-3 gap-4">
        <div className="p-5 rounded-2xl bg-white dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 shadow-xs">
          <span className="text-xs font-semibold text-slate-500 block mb-1">This Month Collected ({currentYearMonth})</span>
          <div className="text-3xl font-extrabold font-mono text-emerald-600 dark:text-emerald-400">
            ₹{totalCollected.toLocaleString('en-IN')}
          </div>
          <span className="text-xs text-slate-400 mt-2 block">
            From {currentMonthInvoices.length} paid receipts this month
          </span>
        </div>

        <div className="p-5 rounded-2xl bg-white dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 shadow-xs">
          <span className="text-xs font-semibold text-slate-500 block mb-1">Outstanding / Overdue Dues</span>
          <div className="text-3xl font-extrabold font-mono text-rose-600 dark:text-rose-400">
            ₹{totalPending.toLocaleString('en-IN')}
          </div>
          <span className="text-xs text-rose-600 mt-2 block font-medium">
            {overdueStudents.length} Overdue · {dueSoonStudents.length} Due in 3 days
          </span>
        </div>

        <div className="p-5 rounded-2xl bg-white dark:bg-slate-900 border border-slate-200/80 dark:border-slate-800 shadow-xs">
          <span className="text-xs font-semibold text-slate-500 block mb-1">Projected Monthly Run-Rate</span>
          <div className="text-3xl font-extrabold font-mono text-slate-900 dark:text-white">
            ₹{(students.reduce((acc, s) => acc + s.monthlyFee, 0)).toLocaleString('en-IN')}
          </div>
          <span className="text-xs text-slate-400 mt-2 block">
            Based on current {students.length} active seat contracts
          </span>
        </div>
      </div>

      {/* Urgent Overdue Members Quick Action Strip */}
      {overdueStudents.length > 0 && (
        <div className="p-5 rounded-2xl bg-rose-50/70 dark:bg-rose-950/30 border border-rose-200/80 dark:border-rose-900/60 space-y-3">
          <div className="flex items-center justify-between">
            <div className="flex items-center gap-2">
              <AlertCircle className="w-5 h-5 text-rose-600" />
              <h3 className="text-sm font-bold text-rose-950 dark:text-rose-200">
                Action Required: {overdueStudents.length} Students Have Overdue Fees
              </h3>
            </div>
            <button
              onClick={onOpenNotifications}
              className="text-xs font-bold text-rose-700 dark:text-rose-400 hover:underline"
            >
              Dispatch Reminders
            </button>
          </div>

          <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-2.5">
            {overdueStudents.slice(0, 6).map(s => {
              const waUrl = generateWhatsAppReminderUrl(s.phone, s.name, s.renewalDueDate, s.monthlyFee, settings.name, s.assignedDesk);
              return (
                <div
                  key={s.id}
                  className="p-3 rounded-xl bg-white dark:bg-slate-900 border border-rose-200/60 dark:border-rose-900/40 flex items-center justify-between gap-2"
                >
                  <div className="min-w-0">
                    <span className="text-xs font-bold text-slate-900 dark:text-white truncate block">
                      {s.name}
                    </span>
                    <span className="text-[11px] text-rose-600 font-mono">
                      Due: {s.renewalDueDate} · Desk {s.assignedDesk} (₹{s.monthlyFee})
                    </span>
                  </div>

                  <div className="flex items-center gap-1.5 shrink-0">
                    <a
                      href={waUrl}
                      target="_blank"
                      rel="noopener noreferrer"
                      className="p-1.5 rounded-lg bg-emerald-50 text-emerald-600 hover:bg-emerald-100"
                      title="WhatsApp Reminder"
                    >
                      <MessageSquare className="w-3.5 h-3.5" />
                    </a>
                    <button
                      onClick={() => {
                        if (!isAdmin) {
                          onRequireAdmin();
                          return;
                        }
                        handleOpenPay(s);
                      }}
                      className="px-2.5 py-1 rounded-lg bg-slate-900 dark:bg-indigo-600 text-white font-semibold text-xs hover:bg-slate-800"
                    >
                      {isAdmin ? 'Collect' : 'Admin'}
                    </button>
                  </div>
                </div>
              );
            })}
          </div>
        </div>
      )}

      {/* Invoices & Payment Ledger Table */}
      <div className="bg-white dark:bg-slate-900 rounded-2xl border border-slate-200/80 dark:border-slate-800 overflow-hidden shadow-xs">
        <div className="p-4 border-b border-slate-100 dark:border-slate-800 flex flex-col sm:flex-row sm:items-center justify-between gap-3">
          <div>
            <h3 className="text-sm font-bold text-slate-900 dark:text-white">
              Fee Invoices & Transactions Ledger
            </h3>
            <p className="text-xs text-slate-500">Official receipt history with printable slips</p>
          </div>

          <div className="relative">
            <Search className="w-3.5 h-3.5 text-slate-400 absolute left-3 top-1/2 -translate-y-1/2" />
            <input
              type="text"
              value={searchInvoiceQuery}
              onChange={e => setSearchInvoiceQuery(e.target.value)}
              placeholder="Search receipt / student..."
              className="pl-8 pr-3 py-1.5 rounded-xl bg-slate-100 dark:bg-slate-800 text-xs focus:outline-none w-52 text-slate-900 dark:text-white"
            />
          </div>
        </div>

        <div className="divide-y divide-slate-100 dark:divide-slate-800">
          {filteredInvoices.slice(0, 15).map(inv => (
            <div key={inv.id} className="p-4 flex flex-col sm:flex-row sm:items-center justify-between gap-3 text-xs">
              <div className="min-w-0">
                <div className="flex items-center gap-2">
                  <span className="font-bold text-slate-900 dark:text-white truncate">{inv.studentName}</span>
                  <span className="font-mono text-slate-400 text-[10px]">{inv.id}</span>
                  <span className="px-1.5 py-0.2 rounded bg-emerald-50 text-emerald-700 text-[10px] font-semibold">
                    Paid via {inv.paymentMethod}
                  </span>
                </div>
                <div className="text-[11px] text-slate-500 font-mono mt-0.5">
                  Ref: {inv.transactionRef} · Period: {inv.periodStart} to {inv.periodEnd}
                </div>
              </div>

              <div className="flex items-center justify-between sm:justify-end gap-3 pl-0 sm:pl-4">
                <div className="text-left sm:text-right">
                  <span className="text-sm font-bold font-mono text-slate-900 dark:text-white block">
                    ₹{inv.amount.toLocaleString('en-IN')}
                  </span>
                  <span className="text-[10px] text-slate-400 font-mono">
                    Paid {inv.paidDate}
                  </span>
                </div>

                <button
                  onClick={() => setReceiptToPrint(inv)}
                  className="px-2.5 py-1.5 rounded-xl border border-slate-200 dark:border-slate-700 hover:bg-slate-50 dark:hover:bg-slate-800 text-slate-700 dark:text-slate-200 font-semibold flex items-center gap-1.5 transition-colors"
                >
                  <Printer className="w-3.5 h-3.5" />
                  <span>Receipt</span>
                </button>
              </div>
            </div>
          ))}
        </div>
      </div>
      </>
      )}

      {/* Pick Student to Collect Fee Modal */}
      {isStudentPickerOpen && (
        <div className="fixed inset-0 z-50 bg-slate-950/80 backdrop-blur-md flex items-center justify-center p-4">
          <div className="bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-800 rounded-3xl w-full max-w-md overflow-hidden shadow-2xl">
            <div className="px-6 py-4 border-b border-slate-100 dark:border-slate-800 flex items-center justify-between">
              <div>
                <h3 className="text-base font-bold text-slate-900 dark:text-white">Select Student for Fee Collection</h3>
                <p className="text-xs text-slate-500">Search any student by name, ID, or desk</p>
              </div>
              <button onClick={() => setIsStudentPickerOpen(false)} className="text-slate-400 hover:text-slate-600">✕</button>
            </div>
            <div className="p-4 border-b border-slate-100 dark:border-slate-800">
              <div className="relative">
                <Search className="w-4 h-4 text-slate-400 absolute left-3 top-1/2 -translate-y-1/2" />
                <input
                  type="text"
                  autoFocus
                  value={studentPickerQuery}
                  onChange={e => setStudentPickerQuery(e.target.value)}
                  placeholder="Type student name, LIB-1001, or desk..."
                  className="w-full pl-9 pr-3 py-2 rounded-xl bg-slate-100 dark:bg-slate-800 text-xs text-slate-900 dark:text-white focus:outline-none focus:ring-2 focus:ring-emerald-500"
                />
              </div>
            </div>
            <div className="max-h-72 overflow-y-auto divide-y divide-slate-100 dark:divide-slate-800">
              {students
                .filter(
                  s =>
                    s.name.toLowerCase().includes(studentPickerQuery.toLowerCase()) ||
                    s.id.toLowerCase().includes(studentPickerQuery.toLowerCase()) ||
                    s.assignedDesk.toLowerCase().includes(studentPickerQuery.toLowerCase())
                )
                .slice(0, 25)
                .map(s => (
                  <button
                    key={s.id}
                    type="button"
                    onClick={() => handleOpenPay(s)}
                    className="w-full px-5 py-3 text-left hover:bg-slate-50 dark:hover:bg-slate-800/60 flex items-center justify-between gap-2 transition-colors"
                  >
                    <div className="min-w-0">
                      <span className="text-xs font-bold text-slate-900 dark:text-white block truncate">
                        {s.name}
                      </span>
                      <span className="text-[11px] text-slate-500 font-mono">
                        {s.id} · Desk {s.assignedDesk} · {formatStudentShifts(s.shifts, s.shift)}
                      </span>
                    </div>
                    <span className="px-2.5 py-1 rounded-lg bg-emerald-50 dark:bg-emerald-950/50 text-emerald-700 dark:text-emerald-300 font-mono text-xs font-bold shrink-0">
                      ₹{s.monthlyFee}
                    </span>
                  </button>
                ))}
            </div>
          </div>
        </div>
      )}

      {/* Record Fee Payment Modal */}
      {selectedStudentForPay && (
        <div className="fixed inset-0 z-50 bg-slate-950/80 backdrop-blur-md flex items-start sm:items-center justify-center p-3 sm:p-4 overflow-y-auto">
          <div className="bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-800 rounded-3xl w-full max-w-md max-h-[92dvh] flex flex-col overflow-hidden shadow-2xl my-auto">
            <div className="px-6 py-4 border-b border-slate-100 dark:border-slate-800 flex items-center justify-between shrink-0">
              <h3 className="text-base font-bold text-slate-900 dark:text-white">Collect Membership Fee</h3>
              <button onClick={() => setSelectedStudentForPay(null)} className="text-slate-400 hover:text-slate-600">✕</button>
            </div>

            <form onSubmit={handleConfirmPayment} className="p-5 sm:p-6 space-y-4 overflow-y-auto flex-1 overscroll-contain">
              <div className="p-3.5 rounded-2xl bg-slate-50 dark:bg-slate-800/60 border border-slate-200 dark:border-slate-700">
                <h4 className="text-sm font-bold text-slate-900 dark:text-white">{selectedStudentForPay.name}</h4>
                <p className="text-xs text-slate-500 font-mono">
                  ID: {selectedStudentForPay.id} · Desk: {selectedStudentForPay.assignedDesk}
                  {targetYearMonthForPay ? ` · Billing Month: ${targetYearMonthForPay}` : ''}
                </p>
                <p className="text-xs text-slate-500 mt-1">
                  Shift: {formatStudentShifts(selectedStudentForPay.shifts, selectedStudentForPay.shift)} · Plan: {selectedStudentForPay.plan.replace('_', ' ')}
                </p>
              </div>

              <div>
                <label className="text-xs font-semibold text-slate-700 dark:text-slate-300 block mb-1">
                  Payment Amount (₹)
                </label>
                <input
                  type="number"
                  required
                  value={paymentAmount}
                  onChange={e => setPaymentAmount(Number(e.target.value))}
                  className="w-full px-3 py-2 rounded-xl border border-slate-200 dark:border-slate-700 bg-white dark:bg-slate-800 text-sm font-mono font-bold"
                />
              </div>

              <div>
                <label className="text-xs font-semibold text-slate-700 dark:text-slate-300 block mb-1.5">
                  Payment Method
                </label>
                <div className="grid grid-cols-3 gap-2">
                  {(['UPI', 'CASH', 'CARD'] as const).map(m => (
                    <button
                      key={m}
                      type="button"
                      onClick={() => setPaymentMethod(m)}
                      className={`py-2 rounded-xl text-xs font-bold transition-colors ${
                        paymentMethod === m
                          ? 'bg-slate-900 dark:bg-indigo-600 text-white shadow-xs'
                          : 'bg-slate-100 dark:bg-slate-800 text-slate-700 dark:text-slate-300'
                      }`}
                    >
                      {m}
                    </button>
                  ))}
                </div>
              </div>

              {paymentMethod === 'UPI' && (
                <PhonePeScannerCard
                  settings={settings}
                  amount={paymentAmount}
                  note={`${selectedStudentForPay.id} Fee`}
                  compact
                />
              )}

              <div className="p-3 bg-emerald-50 dark:bg-emerald-950/40 rounded-xl text-xs text-emerald-800 dark:text-emerald-300">
                ✓ Extends student membership through next 30 days and logs official tax receipt.
              </div>

              <div className="pt-2 flex justify-end gap-2">
                <button
                  type="button"
                  onClick={() => setSelectedStudentForPay(null)}
                  className="px-4 py-2 text-xs font-semibold text-slate-600"
                >
                  Cancel
                </button>
                <button
                  type="submit"
                  className="px-5 py-2.5 rounded-xl bg-emerald-600 hover:bg-emerald-700 text-white font-bold text-xs shadow-sm flex items-center gap-1.5"
                >
                  <CheckCircle2 className="w-4 h-4" />
                  <span>Confirm & Issue Receipt</span>
                </button>
              </div>
            </form>
          </div>
        </div>
      )}

      {/* Printable Thermal Receipt Modal */}
      {receiptToPrint && (
        <div className="fixed inset-0 z-50 bg-slate-950/80 backdrop-blur-md flex items-center justify-center p-4">
          <div className="bg-white text-slate-900 border border-slate-200 rounded-3xl w-full max-w-sm overflow-hidden shadow-2xl">
            <div className="p-6 font-mono text-xs space-y-4">
              <div className="text-center border-b border-dashed border-slate-300 pb-4">
                <h3 className="font-bold text-base tracking-tight font-sans">{settings.name}</h3>
                <p className="text-[11px] text-slate-500">{settings.tagline}</p>
                <p className="text-[10px] text-slate-400 mt-1">{settings.address}</p>
                <p className="text-[10px] text-slate-400">Tel: {settings.phone}</p>
              </div>

              <div className="space-y-1 text-[11px]">
                <div className="flex justify-between">
                  <span className="text-slate-500">Receipt No:</span>
                  <span className="font-bold">{receiptToPrint.id}</span>
                </div>
                <div className="flex justify-between">
                  <span className="text-slate-500">Date & Time:</span>
                  <span>{receiptToPrint.paidDate}</span>
                </div>
                <div className="flex justify-between">
                  <span className="text-slate-500">Member Name:</span>
                  <span className="font-bold">{receiptToPrint.studentName}</span>
                </div>
                <div className="flex justify-between">
                  <span className="text-slate-500">Student ID:</span>
                  <span>{receiptToPrint.studentId}</span>
                </div>
                <div className="flex justify-between">
                  <span className="text-slate-500">Payment Mode:</span>
                  <span>{receiptToPrint.paymentMethod}</span>
                </div>
                <div className="flex justify-between">
                  <span className="text-slate-500">Txn Ref:</span>
                  <span>{receiptToPrint.transactionRef}</span>
                </div>
              </div>

              <div className="border-t border-b border-dashed border-slate-300 py-2 space-y-1">
                <div className="flex justify-between">
                  <span>Monthly Study Slot Pass</span>
                  <span className="font-bold">₹{receiptToPrint.amount.toLocaleString('en-IN')}</span>
                </div>
                <div className="flex justify-between text-[10px] text-slate-500">
                  <span>Valid: {receiptToPrint.periodStart} to {receiptToPrint.periodEnd}</span>
                </div>
              </div>

              <div className="flex justify-between text-sm font-bold pt-1">
                <span>TOTAL PAID:</span>
                <span>₹{receiptToPrint.amount.toLocaleString('en-IN')}</span>
              </div>

              <div className="text-center text-[10px] text-slate-400 pt-2">
                Thank you for studying at {settings.name}! Maintain silence in the reading halls.
              </div>
            </div>

            <div className="p-4 bg-slate-50 border-t border-slate-100 flex items-center justify-between no-print">
              <button
                onClick={() => setReceiptToPrint(null)}
                className="px-4 py-2 text-xs font-semibold text-slate-600"
              >
                Close
              </button>
              <button
                onClick={() => window.print()}
                className="px-4 py-2 rounded-xl bg-slate-900 text-white text-xs font-bold flex items-center gap-1.5"
              >
                <Printer className="w-3.5 h-3.5" />
                <span>Print Receipt</span>
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
};
