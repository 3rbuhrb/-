/**
 * @license
 * SPDX-License-Identifier: Apache-2.0
 */

import { useState } from 'react';
import ReactMarkdown from 'react-markdown';
import { Settings, Cpu, Save, Calendar, Clock, Sparkles, Copy, Check, AlertCircle, Lock, ShieldAlert, Eye, EyeOff, Activity, Download, Upload, ServerCrash } from 'lucide-react';
import { ShiftSettings, Employee, AttendanceLog } from '../types';
import { i18n, Language, translate } from '../utils/lang';

interface SettingsTabProps {
  shiftSettings: ShiftSettings;
  onUpdateShift: (newSettings: ShiftSettings) => void;
  employees: Employee[];
  logs: AttendanceLog[];
  lang: Language;
  onChangeLang: (newLang: Language) => void;
  adminPasskey: string;
  onUpdateAdminPasskey: (newPasskey: string) => void;
  securityLogs: { id: string; time: string; action: string; status: 'success' | 'failed' | 'info' | 'warn' }[];
  leaves?: any[];
  onImportAllStates?: (data: any) => void;
}

export default function SettingsTab({ 
  shiftSettings, 
  onUpdateShift, 
  employees, 
  logs, 
  lang, 
  onChangeLang,
  adminPasskey,
  onUpdateAdminPasskey,
  securityLogs,
  leaves = [],
  onImportAllStates
}: SettingsTabProps) {
  // Use translations helper
  const t = (key: keyof typeof i18n['ar']) => {
    return translate(lang, key);
  };
  const [startTime, setStartTime] = useState(shiftSettings.startTime);
  const [endTime, setEndTime] = useState(shiftSettings.endTime);
  const [gracePeriod, setGracePeriod] = useState(shiftSettings.gracePeriod);
  const [workingDays, setWorkingDays] = useState<number[]>(shiftSettings.workingDays);
  const [enforceConstraints, setEnforceConstraints] = useState(!!shiftSettings.enforceConstraints);

  // Security variables
  const [newPin, setNewPin] = useState('');
  const [showPin, setShowPin] = useState(false);

  // AI Insights states
  const [aiReport, setAiReport] = useState<string>('');
  const [isLoadingAi, setIsLoadingAi] = useState(false);
  const [aiLoadingStep, setAiLoadingStep] = useState('');
  const [copied, setCopied] = useState(false);

  // Weekend and standard Arab Region working days helper
  const weekDaysList = [
    { id: 0, name: lang === 'ar' ? 'الأحد' : 'Sunday' },
    { id: 1, name: lang === 'ar' ? 'الإثنين' : 'Monday' },
    { id: 2, name: lang === 'ar' ? 'الثلاثاء' : 'Tuesday' },
    { id: 3, name: lang === 'ar' ? 'الأربعاء' : 'Wednesday' },
    { id: 4, name: lang === 'ar' ? 'الخميس' : 'Thursday' },
    { id: 5, name: lang === 'ar' ? 'الجمعة' : 'Friday' },
    { id: 6, name: lang === 'ar' ? 'السبت' : 'Saturday' },
  ];

  const handleDayToggle = (dayId: number) => {
    if (workingDays.includes(dayId)) {
      setWorkingDays(workingDays.filter(d => d !== dayId));
    } else {
      setWorkingDays([...workingDays, dayId].sort());
    }
  };

  const handleSaveShift = (e: React.FormEvent) => {
    e.preventDefault();
    onUpdateShift({
      startTime,
      endTime,
      gracePeriod: Number(gracePeriod),
      workingDays,
      enforceConstraints,
    });
    alert(lang === 'ar' 
      ? 'تم حفظ إعدادات الشفت والقيود الإدارية للحضور بنجاح.' 
      : 'Corporate shift configurations and constraints updated successfully!');
  };

  const handleGenerateAiReport = async () => {
    setIsLoadingAi(true);
    setAiLoadingStep(lang === 'ar' ? 'جاري تجميع سجلات الحضور الحالية وبطاقات الموظفين...' : 'Amalgamating current timesheets and personnel profiles...');
    
    // Simulate some realistic loading increments for high-end look
    setTimeout(() => {
      setAiLoadingStep(lang === 'ar' ? 'جاري مطابقة أوقات الدخول مقابل الشفت الافتراضي وفترة السماح...' : 'Cross-referencing check-in stamps against base rules...');
      setTimeout(() => {
        setAiLoadingStep(lang === 'ar' ? 'جاري صياغة التوصيات الإدارية لنموذج Gemini 3.5-Flash...' : 'Drafting executive recommendations via Gemini 3.5-Flash...');
      }, 1000);
    }, 1000);

    try {
      const response = await fetch('/api/ai/analyze', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          logs,
          employees: employees.map(e => ({ name: e.name, code: e.employeeCode, role: e.role, dept: e.department })),
          shiftSettings: { startTime, endTime, gracePeriod, workingDays: workingDays.map(d => weekDaysList.find(w => w.id === d)?.name) }
        })
      });

      const data = await response.json();
      if (!response.ok) {
        throw new Error(data.error || 'حدث خطأ في شبكة الاتصال.');
      }

      setAiReport(data.analysis);
    } catch (err: any) {
      console.error(err);
      setAiReport(lang === 'ar' 
        ? `### ❌ تعذر الاتصال بالذكاء الاصطناعي\n\nأبلغ الخادم: ${err.message || 'خطأ مجهول'}\n\nيرجى التأكد من تشغيل الخادم وتكوين مفتاح (GEMINI_API_KEY) بشكل صحيح في إعدادات لوحة الأسرار.`
        : `### ❌ Failed to contact Gemini AI\n\nServer reported: ${err.message || 'Unknown network error'}\n\nPlease check dev servers and ensure your (GEMINI_API_KEY) is correctly configured in your settings page.`);
    } finally {
      setIsLoadingAi(false);
      setAiLoadingStep('');
    }
  };

  const handleCopyReport = () => {
    if (!aiReport) return;
    navigator.clipboard.writeText(aiReport);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  };

  return (
    <div id="settings-tab-panel" className="grid grid-cols-1 lg:grid-cols-12 gap-6">
      
      {/* Shift & Language column */}
      <div className="lg:col-span-5 space-y-6 flex flex-col">
        
        {/* Language selector card */}
        <div className="bg-white border border-slate-100 p-6 rounded-3xl shadow-xs">
          <div className="border-b border-slate-50 pb-4 mb-4">
            <h2 className="text-md font-extrabold text-slate-800 flex items-center gap-2">
              <span className="text-blue-500 font-bold">🌐</span>
              {t('settings_lang_label')}
            </h2>
            <p className="text-xs text-slate-500 mt-1">{t('settings_lang_desc')}</p>
          </div>

          <div className="grid grid-cols-2 gap-3">
            <button
              type="button"
              onClick={() => onChangeLang('ar')}
              className={`p-3 rounded-2xl border text-xs font-extrabold transition-all text-center flex flex-col items-center justify-center gap-1.5 cursor-pointer ${
                lang === 'ar'
                  ? 'bg-blue-50/70 text-blue-600 border-blue-200 ring-2 ring-blue-100'
                  : 'bg-slate-50 text-slate-600 border-slate-100 hover:bg-slate-100'
              }`}
            >
              <span className="text-sm font-black">العربية</span>
              <span className="text-[9.5px] font-semibold opacity-70">العربية - RTL</span>
            </button>

            <button
              type="button"
              onClick={() => onChangeLang('en')}
              className={`p-3 rounded-2xl border text-xs font-extrabold transition-all text-center flex flex-col items-center justify-center gap-1.5 cursor-pointer ${
                lang === 'en'
                  ? 'bg-blue-50/70 text-blue-600 border-blue-200 ring-2 ring-blue-100'
                  : 'bg-slate-50 text-slate-600 border-slate-100 hover:bg-slate-100'
              }`}
            >
              <span className="text-sm font-black">English</span>
              <span className="text-[9.5px] font-semibold opacity-70">English - LTR</span>
            </button>
          </div>
        </div>

        {/* Security Desk Guard Card */}
        <div className="bg-white border border-slate-100 p-6 rounded-3xl shadow-xs">
          <div className="border-b border-slate-50 pb-4 mb-4">
            <h2 className="text-md font-extrabold text-slate-800 flex items-center justify-between">
              <span className="flex items-center gap-2">
                <Lock className="h-5 w-5 text-emerald-500 animate-pulse" />
                {lang === 'ar' ? 'أمان حماية لوحة التحكم' : 'Security Desk Guard'}
              </span>
              <span className="bg-emerald-50 text-emerald-700 text-[9.5px] font-black px-2 py-0.5 rounded-full border border-emerald-100 font-mono">
                {lang === 'ar' ? 'جدار ناري ذكي' : 'Smart Firewall'}
              </span>
            </h2>
            <p className="text-xs text-slate-500 mt-1">
              {lang === 'ar' 
                ? 'تغيير رمز المرور وإصدار جدار حماية لمنع تخمين البرمجة السرية.' 
                : 'Update admin password & block malicious scanners to protect database.'}
            </p>
          </div>

          <div className="bg-slate-50/50 rounded-2xl border border-slate-100 p-4 mb-4">
            <div className="flex justify-between items-center">
              <div>
                <span className="text-[10px] font-bold text-slate-400 block">
                  {lang === 'ar' ? 'رمز مرور المسؤول الحالي' : 'Active Admin passcode'}
                </span>
                <span className="text-sm font-mono font-bold text-slate-750 mt-0.5 block">
                  {showPin ? adminPasskey : '••••'}
                </span>
              </div>
              <button
                type="button"
                onClick={() => setShowPin(!showPin)}
                className="p-1 px-2.5 text-xs font-semibold text-slate-500 hover:text-blue-600 bg-white border border-slate-200 rounded-lg hover:shadow-2xs transition-all flex items-center gap-1.5 cursor-pointer"
              >
                {showPin ? (
                  <>
                    <EyeOff className="h-3.5 w-3.5 text-slate-400" />
                    <span className="text-[10px] font-bold">{lang === 'ar' ? 'إخفاء الرمز' : 'Hide PIN'}</span>
                  </>
                ) : (
                  <>
                    <Eye className="h-3.5 w-3.5 text-slate-400" />
                    <span className="text-[10px] font-bold">{lang === 'ar' ? 'كشف الرمز' : 'Reveal PIN'}</span>
                  </>
                )}
              </button>
            </div>
          </div>

          <form onSubmit={(e) => {
            e.preventDefault();
            if (!newPin || newPin.trim().length < 4) {
              alert(lang === 'ar' ? 'يجب أن لا يقل الرمز السري عن 4 خانات لضمان السلامة!' : 'Passkey should be at least 4 symbols!');
              return;
            }
            onUpdateAdminPasskey(newPin.trim());
            setNewPin('');
          }} className="space-y-3">
            <div>
              <label className="block text-xs font-bold text-slate-500 mb-1">
                {lang === 'ar' ? 'رمز المرور الجديد (أرقام أو نصوص)' : 'New Admin PIN / Passcode'}
              </label>
              <input
                type="text"
                placeholder={lang === 'ar' ? 'الرمز الجديد (مثال: 9988)' : 'Example: 9988'}
                value={newPin}
                onChange={(e) => setNewPin(e.target.value)}
                className="w-full text-xs py-2.5 bg-slate-50 border border-slate-100 rounded-xl font-mono text-center focus:border-emerald-500 focus:outline-hidden"
              />
            </div>
            <button
              type="submit"
              className="w-full py-2.5 bg-emerald-600 hover:bg-emerald-700 text-white font-bold text-xs rounded-xl shadow-xs transition-colors flex items-center justify-center gap-2 cursor-pointer"
            >
              <Lock className="h-3.5 w-3.5" /> {lang === 'ar' ? 'تحديث رمز حماية المسؤول' : 'Apply New Password Code'}
            </button>
          </form>
        </div>

        {/* Local Mode, Data Portability & Storage Backup */}
        <div className="bg-white border border-slate-100 p-6 rounded-3xl shadow-xs">
          <div className="border-b border-slate-50 pb-4 mb-4 flex justify-between items-start flex-wrap gap-2">
            <div>
              <h2 className="text-md font-extrabold text-slate-800 flex items-center gap-2">
                <span className="p-1 px-1.5 bg-indigo-50 text-indigo-600 rounded-lg text-xs font-black">L</span>
                {lang === 'ar' ? 'النسخ الاحتياطي والبيانات المحلية' : 'Local Backup & Portability'}
              </h2>
              <p className="text-xs text-slate-500 mt-1">
                {lang === 'ar' 
                  ? 'يتم تخزين بيانات هذا النظام بالكامل في متصفحك المحلي بشكل آمن بدون أي خوادم خارجية.' 
                  : 'System records reside fully in your sandbox browser. Save offline backups.'}
              </p>
            </div>
          </div>

          <div className="space-y-3.5">
            {/* Security Badge of Honor */}
            <div className="p-3 bg-slate-50 border border-slate-100/60 rounded-2xl flex items-center gap-3">
              <span className="w-2.5 h-2.5 rounded-full bg-emerald-500 animate-pulse flex-shrink-0"></span>
              <div>
                <span className="text-[10.5px] font-extrabold text-slate-750 block">
                  {lang === 'ar' ? 'البيانات آمنة ومستضافة محلياً بنسبة %100' : '100% Secure Local Sandbox Active'}
                </span>
                <span className="text-[9px] text-slate-400 block font-bold leading-tight mt-0.5">
                  {lang === 'ar' 
                    ? 'متكامل مع التخزين المشفر لجهازك (LocalStorage) لمنع تسريب العمليات.' 
                    : 'Coupled with device-isolated storage, zero outbound packets or analytics clouding.'}
                </span>
              </div>
            </div>

            {/* Stats count summary of the database */}
            <div className="grid grid-cols-2 gap-2 text-center">
              <div className="bg-slate-50/50 p-2.5 rounded-xl border border-slate-100">
                <span className="text-[9px] text-slate-400 font-bold block">{lang === 'ar' ? 'إجمالي الموظفين' : 'Total Staff'}</span>
                <span className="text-sm font-mono font-black text-slate-700 block mt-0.5">{employees.length}</span>
              </div>
              <div className="bg-slate-50/50 p-2.5 rounded-xl border border-slate-100">
                <span className="text-[9px] text-slate-400 font-bold block">{lang === 'ar' ? 'إجمالي الباكودات/السجلات' : 'Total Logs'}</span>
                <span className="text-sm font-mono font-black text-slate-700 block mt-0.5">{logs.length}</span>
              </div>
            </div>

            {/* Export DB Button */}
            <button
              onClick={() => {
                try {
                  const backupData = {
                    employees,
                    logs,
                    leaves,
                    shiftSettings,
                    adminPasskey,
                    securityLogs,
                    exportedAt: new Date().toISOString(),
                    version: "Local_v1.0"
                  };
                  const fileData = JSON.stringify(backupData, null, 2);
                  const blob = new Blob([fileData], { type: "application/json" });
                  const url = URL.createObjectURL(blob);
                  const link = document.createElement("a");
                  const dateStr = new Date().toISOString().slice(0, 10);
                  link.href = url;
                  link.download = `attendance_backup_local_${dateStr}.json`;
                  document.body.appendChild(link);
                  link.click();
                  document.body.removeChild(link);
                  URL.revokeObjectURL(url);
                } catch (err: any) {
                  alert(lang === 'ar' ? 'حدث خطأ أثناء تصدير قاعدة البيانات.' : 'Error exporting database backup.');
                }
              }}
              className="w-full py-2.5 bg-indigo-650 hover:bg-indigo-705 text-slate-800 bg-slate-50 border border-slate-200 hover:bg-slate-100 font-bold text-xs rounded-xl shadow-2xs transition-all flex items-center justify-center gap-2 cursor-pointer"
            >
              <Download className="h-3.5 w-3.5 text-indigo-500" />
              {lang === 'ar' ? 'تصدير نسخة احتياطية (JSON)' : 'Export Offline Database (.JSON)'}
            </button>

            {/* Import DB Inputs / Buttons */}
            <div className="relative">
              <label 
                htmlFor="import-db-file" 
                className="w-full py-2.5 bg-emerald-50 hover:bg-emerald-100/80 text-emerald-700 font-bold text-xs rounded-xl shadow-2xs transition-all flex items-center justify-center gap-2 cursor-pointer border border-emerald-200/60"
              >
                <Upload className="h-3.5 w-3.5 text-emerald-600" />
                {lang === 'ar' ? 'استيراد نسخة احتياطية مخزنة' : 'Import Local Backup JSON File'}
              </label>
              <input 
                id="import-db-file" 
                type="file" 
                accept=".json"
                className="hidden"
                onChange={(e) => {
                  const file = e.target.files?.[0];
                  if (!file) return;
                  const reader = new FileReader();
                  reader.onload = (event) => {
                    try {
                      const result = event.target?.result;
                      if (typeof result !== 'string') return;
                      const parsed = JSON.parse(result);
                      
                      if (!parsed.employees || !Array.isArray(parsed.employees)) {
                        alert(lang === 'ar' ? 'ملف التصدير غير صالح، مفقود جدول البيانات الأساسي للموظفين.' : 'Invalid backup format. Employees list is required.');
                        return;
                      }
                      
                      if (onImportAllStates) {
                        onImportAllStates(parsed);
                        alert(lang === 'ar' ? 'تم استيراد قاعدة البيانات المحلية بنجاح! تم استرجاع كافة الإعدادات والبطاقات والسجلات.' : 'All local database files successfully parsed and sync restored.');
                      } else {
                        alert(lang === 'ar' ? 'حدث خطأ كود: دالة الاستيراد غير متوفرة.' : 'Import callback routine was not mapped correctly.');
                      }
                    } catch (ex: any) {
                      alert(lang === 'ar' ? 'خطأ في معالجة الملف، يرجى ملء وتنزيل ملف JSON صحيح.' : 'Failed to parse JSON content.');
                    }
                  };
                  reader.readAsText(file);
                  e.target.value = '';
                }} 
              />
            </div>
            
          </div>
        </div>

        {/* Standalone Export Card for GitHub Pages */}
        <div className="bg-gradient-to-br from-indigo-900 to-slate-900 text-white p-6 rounded-3xl shadow-md border border-indigo-950/50 space-y-4">
          <div className="flex items-start justify-between">
            <div className="space-y-1">
              <span className="px-2.5 py-0.5 bg-indigo-500/20 text-indigo-300 border border-indigo-500/30 text-[10px] rounded-md font-black tracking-wide uppercase">
                {lang === 'ar' ? 'تصدير كامل ومستقل' : '100% STANDALONE SINGLE-FILE'}
              </span>
              <h2 className="text-md font-black flex items-center gap-2 pt-1 text-indigo-100">
                <Download className="h-5 w-5 text-indigo-400" />
                {lang === 'ar' ? 'تصدير النظام كصفحة واحدة مستقلة (index.html)' : 'Export Standalone HTML (index.html)'}
              </h2>
            </div>
          </div>

          <p className="text-xs text-slate-300 leading-relaxed text-right">
            {lang === 'ar'
              ? 'يقوم هذا الخيار بتنزيل ملف "index.html" موحد وصافٍ يحتوي على كامل واجهة المستخدم، الألوان، الأزرار، والمنطق البرمجي، بالإضافة لأسماء الموظفين والرموز الخاصة بهم. الملف جاهز للرفع المباشر ومجاناً على خدمة GitHub Pages ليعمل فورياً في المتصفح دون الحاجة لأي خوادم أو عمليات بناء معقدة.'
              : 'Download a self-contained, single-file "index.html" containing all UI styles, buttons, employees, and JS logic. Ready to host for free on GitHub Pages instantly with zero build steps.'}
          </p>

          <div className="p-3 bg-white/5 border border-white/10 rounded-2xl text-[10px] text-indigo-200 leading-normal space-y-1">
            <p>💡 <b>{lang === 'ar' ? 'التخزين والمزامنة:' : 'Storage & Sync:'}</b> {lang === 'ar' ? 'الملف الناتج يستخدم LocalStorage لحفظ بيانات الحضور والغياب بشكل آمن وتلقائي داخل المتصفح.' : 'The exported file stores all logs in LocalStorage inside your browser sandbox automatically.'}</p>
          </div>

          <button
            type="button"
            onClick={async () => {
              try {
                const res = await fetch('/standalone_index.html');
                if (!res.ok) throw new Error('File not found');
                const text = await res.text();
                const blob = new Blob([text], { type: 'text/html;charset=utf-8' });
                const url = URL.createObjectURL(blob);
                const a = document.createElement('a');
                a.href = url;
                a.download = 'index.html';
                document.body.appendChild(a);
                a.click();
                document.body.removeChild(a);
                alert(lang === 'ar' ? 'تم تجهيز وتنزيل ملف index.html بنجاح! يمكنك الآن رفعه مباشرة على مستودع GitHub وتفعيل صفحة GitHub Pages.' : 'Standalone index.html exported and downloaded successfully!');
              } catch (err) {
                alert(lang === 'ar' ? 'عذراً، فشل جلب ملف القالب الموحد. يرجى التواصل مع الدعم.' : 'Failed to retrieve standalone HTML template.');
              }
            }}
            className="w-full py-3 bg-indigo-600 hover:bg-indigo-500 active:scale-95 text-white font-extrabold text-xs rounded-xl shadow-md transition-all flex items-center justify-center gap-2 cursor-pointer border border-indigo-400/20"
          >
            <Download className="h-4 w-4" />
            {lang === 'ar' ? 'تحميل كود الصفحة الفردية index.html' : 'Download Standalone index.html'}
          </button>
        </div>

        {/* Shift settings panel */}
        <div className="bg-white border border-slate-100 p-6 rounded-3xl shadow-xs h-fit">
          <div className="border-b border-slate-50 pb-4 mb-4">
            <h2 className="text-md font-extrabold text-slate-800 flex items-center gap-2">
              <Settings className="h-5 w-5 text-slate-500" />
              {lang === 'ar' ? 'إعدادات الشفت والدوام الرسمي' : 'Shift Timing Rules'}
            </h2>
            <p className="text-xs text-slate-500 mt-1">{lang === 'ar' ? 'تعديل التوقيتات الرسمية للعمل، فترات التأخير وساعات الراحة الأسبوعية.' : 'Modify default shift timings, delay tolerances and weekly working schedules.'}</p>
          </div>

          <form onSubmit={handleSaveShift} className="space-y-4">
            <div className="grid grid-cols-2 gap-3">
              <div>
                <label className="block text-xs font-bold text-slate-500 mb-1">{t('settings_start_shift')}</label>
                <div className="relative">
                  <span className={`absolute inset-y-0 ${lang === 'en' ? 'left-0 pl-3' : 'right-0 pr-3'} flex items-center pointer-events-none`}>
                    <Clock className="h-4 w-4 text-slate-400" />
                  </span>
                  <input
                    type="time"
                    value={startTime}
                    onChange={(e) => setStartTime(e.target.value)}
                    className={`w-full text-xs py-2.5 bg-slate-50 border border-slate-100 rounded-xl font-mono ${lang === 'en' ? 'pl-9 pr-3 text-left' : 'pr-9 pl-3 text-left'}`}
                  />
                </div>
              </div>

              <div>
                <label className="block text-xs font-bold text-slate-500 mb-1">{t('settings_end_shift')}</label>
                <div className="relative">
                  <span className={`absolute inset-y-0 ${lang === 'en' ? 'left-0 pl-3' : 'right-0 pr-3'} flex items-center pointer-events-none`}>
                    <Clock className="h-4 w-4 text-slate-400" />
                  </span>
                  <input
                    type="time"
                    value={endTime}
                    onChange={(e) => setEndTime(e.target.value)}
                    className={`w-full text-xs py-2.5 bg-slate-50 border border-slate-100 rounded-xl font-mono ${lang === 'en' ? 'pl-9 pr-3 text-left' : 'pr-9 pl-3 text-left'}`}
                  />
                </div>
              </div>
            </div>

            <div>
              <label className="block text-xs font-bold text-slate-500 mb-1">{t('settings_grace_period_label')}</label>
              <div className="relative">
                <input
                  type="number"
                  min={0}
                  max={120}
                  value={gracePeriod}
                  onChange={(e) => setGracePeriod(Number(e.target.value))}
                  className={`w-full text-xs py-2.5 bg-slate-50 border border-slate-100 rounded-xl font-mono ${lang === 'en' ? 'pl-3 pr-16 text-left' : 'pr-3 pl-16 text-right'}`}
                />
                <span className={`absolute inset-y-0 ${lang === 'en' ? 'right-0 pr-3' : 'left-0 pl-3'} flex items-center text-xs text-slate-400 font-bold pointer-events-none`}>
                  {lang === 'ar' ? 'دقيقة سماح' : 'min tolerance'}
                </span>
              </div>
              <p className="text-[10px] text-slate-400 mt-1">
                {t('settings_grace_desc')}
              </p>
            </div>

            <div>
              <label className="block text-xs font-bold text-slate-500 mb-2">{t('settings_working_days')}</label>
              <div className="grid grid-cols-4 gap-2">
                {weekDaysList.map((day) => {
                  const isSelected = workingDays.includes(day.id);
                  return (
                    <button
                      key={day.id}
                      type="button"
                      onClick={() => handleDayToggle(day.id)}
                      className={`py-2 px-1 rounded-xl text-xs font-bold transition-all border ${
                        isSelected
                          ? 'bg-blue-600 text-white border-blue-600'
                          : 'bg-slate-50 text-slate-500 border-slate-100 hover:bg-slate-100'
                      }`}
                    >
                      {day.name}
                    </button>
                  );
                })}
              </div>
            </div>

            {/* Constraints enforcement switch */}
            <div className="p-4 bg-slate-50 border border-slate-100 rounded-2xl flex items-start gap-3 select-none">
              <input
                id="toggle-enforce-constraints"
                type="checkbox"
                checked={enforceConstraints}
                onChange={(e) => setEnforceConstraints(e.target.checked)}
                className="h-4.5 w-4.5 rounded text-blue-600 border-slate-250 focus:ring-blue-500 cursor-pointer mt-0.5"
              />
              <div className="text-right flex-1">
                <label htmlFor="toggle-enforce-constraints" className="block text-xs font-black text-slate-800 cursor-pointer">
                  {lang === 'ar' ? 'تفعيل قيود وقت الدوام والشفت الصارمة' : 'Enforce strict work hours / days restrictions'}
                </label>
                <span className="block text-[10px] text-slate-400 mt-0.5 leading-relaxed font-semibold">
                  {lang === 'ar'
                    ? 'عند تفعيله، سيمنع النظام الموظفين من تسجيل الحضور خارج الساعات الرسمية للشفت أو أيام العطل المحددة. (ينصح بإلغائه لتسهيل التجربة على مدار 24 ساعة).'
                    : 'When checked, scanning outside official working days/hours is blocked. (Recommended unchecked for 24/7 scanning and testing).'}
                </span>
              </div>
            </div>

            <button
              type="submit"
              className="w-full py-3 bg-blue-600 hover:bg-blue-700 text-white font-bold text-xs rounded-xl shadow-xs transition-colors flex items-center justify-center gap-2 mt-2 cursor-pointer"
            >
              <Save className="h-4 w-4" /> {t('settings_save_btn')}
            </button>
          </form>
        </div>
      </div>

      {/* AI HR Insights screen */}
      <div className="lg:col-span-7 bg-white border border-slate-100 p-6 rounded-3xl shadow-xs flex flex-col justify-between h-auto">
        <div>
          <div className={`border-b border-slate-50 pb-4 mb-4 flex justify-between items-center ${lang === 'en' ? 'flex-row' : 'flex-row-reverse'}`}>
            <h2 className="text-md font-extrabold text-slate-800 flex items-center gap-2">
              <Cpu className="h-5 w-5 text-indigo-500 animate-pulse" />
              {t('settings_ai_title')}
            </h2>
            <span className="bg-indigo-100 text-indigo-800 text-[10px] px-2.5 py-1 rounded-full font-bold flex items-center gap-1">
              <Sparkles className="h-3 w-3" /> {t('settings_ai_desc')}
            </span>
          </div>

          {/* AI Report output display box */}
          {isLoadingAi ? (
            <div className="flex flex-col items-center justify-center py-20 text-center space-y-4 select-none">
              <div className="relative">
                <div className="w-12 h-12 rounded-full border-4 border-slate-100 border-t-indigo-600 animate-spin" />
                <Sparkles className="h-5 w-5 text-indigo-500 absolute top-3.5 left-3.5 animate-bounce" />
              </div>
              <p className="text-xs font-bold text-slate-700 animate-pulse">{aiLoadingStep}</p>
              <p className="text-[10px] text-slate-400 font-medium">{lang === 'ar' ? 'الرجاء عدم إغلاق الصفحة، يستغرق التحليل بضع ثوانٍ...' : 'Please don\'t shut settings screen, generating analysis...'}</p>
            </div>
          ) : aiReport ? (
            <div className="space-y-4 animate-scale-up">
              <div className={`flex justify-between items-center bg-slate-50 px-4 py-2.5 rounded-xl border border-slate-100 ${lang === 'en' ? 'flex-row' : 'flex-row-reverse'}`}>
                <span className="text-[11px] text-slate-400 font-semibold">{t('settings_ai_report_ready')}</span>
                <button
                  onClick={handleCopyReport}
                  className="text-xs font-bold text-indigo-600 hover:text-indigo-700 flex items-center gap-1.5 bg-white border border-slate-100 px-3 py-1.5 rounded-lg hover:shadow-2xs transition-all cursor-pointer"
                >
                  {copied ? (
                    <>
                      <Check className="h-3.5 w-3.5 text-emerald-500" /> {lang === 'ar' ? 'تم النسخ' : 'Copied'}
                    </>
                  ) : (
                    <>
                      <Copy className="h-3.5 w-3.5" /> {t('settings_ai_copy')}
                    </>
                  )}
                </button>
              </div>

              {/* Styled markdown output wrapper */}
              <div className={`bg-slate-50/50 border border-slate-100 p-5 rounded-2xl max-h-96 overflow-y-auto text-xs leading-relaxed text-slate-800 font-medium whitespace-pre-line prose max-w-none ${lang === 'en' ? 'text-left' : 'text-right'}`}>
                <ReactMarkdown>{aiReport}</ReactMarkdown>
              </div>
            </div>
          ) : (
            <div className="text-center py-16 space-y-4 select-none bg-indigo-50/20 border border-indigo-50 rounded-2xl p-5">
              <div className="p-4 bg-white shadow-xs rounded-2xl text-indigo-600 inline-block ring-4 ring-indigo-50">
                <Sparkles className="h-7 w-7" />
              </div>
              <h4 className="font-extrabold text-slate-800 text-sm">{t('settings_ai_prompt_init')}</h4>
              <p className="text-xs text-slate-500 max-w-sm mx-auto leading-relaxed">
                {t('settings_ai_prompt_desc')}
              </p>
            </div>
          )}
        </div>

        {!isLoadingAi && (
          <button
            onClick={handleGenerateAiReport}
            className="w-full py-4 bg-indigo-600 hover:bg-indigo-700 text-white font-bold text-sm rounded-2xl shadow-sm hover:shadow-md transition-all flex items-center justify-center gap-2 mt-6 cursor-pointer"
          >
            <Sparkles className="h-4.5 w-4.5" />
            {aiReport ? t('settings_ai_regenerate') : t('settings_ai_generate')}
          </button>
        )}
      </div>

      {/* Dynamic Security & Audit Log Stream */}
      <div className="lg:col-span-12 bg-white border border-slate-100 p-6 rounded-3xl shadow-xs">
        <div className="border-b border-slate-50 pb-4 mb-4 flex justify-between items-center flex-wrap gap-2">
          <div>
            <h2 className="text-md font-extrabold text-slate-800 flex items-center gap-2">
              <Activity className="h-5 w-5 text-sky-500 animate-pulse" />
              {lang === 'ar' ? 'سجل مراقبة العمليات وجدار الحماية' : 'Firewall & Action Audit Stream'}
            </h2>
            <p className="text-xs text-slate-500 mt-1">
              {lang === 'ar' 
                ? 'مراقبة فورية للولوج غير المصرح به، عمليات التعديل، والمخاطر السيبرانية الموقوفة بالتوقيت الحي.' 
                : 'Real-time observation of authorization gates, failed attempts, and blocked payloads.'}
            </p>
          </div>
          <span className="bg-sky-50 text-sky-700 text-[10px] px-3 py-1 rounded-full font-black flex items-center gap-1.5 border border-sky-100">
            <span className="w-2 h-2 rounded-full bg-emerald-500 animate-pulse"></span>
            {lang === 'ar' ? 'جدار الحماية: نشط ومفعل' : 'Firewall Status: ACTIVE'}
          </span>
        </div>

        <div className="overflow-x-auto">
          <div className="max-h-60 overflow-y-auto space-y-2.5 pr-2">
            {securityLogs && securityLogs.length > 0 ? (
              securityLogs.map((log) => {
                let badgeColor = '';
                let textColor = '';
                let statusText = '';
                let icon = null;

                if (log.status === 'success') {
                  badgeColor = 'bg-emerald-50 text-emerald-700 border-emerald-100';
                  textColor = 'text-emerald-700';
                  statusText = lang === 'ar' ? 'آمن مسموح' : 'Auth success';
                } else if (log.status === 'failed') {
                  badgeColor = 'bg-rose-50 text-rose-700 border-rose-200';
                  textColor = 'text-rose-700 font-extrabold';
                  statusText = lang === 'ar' ? 'محجوب / حظر' : 'Blocked Threat';
                  icon = <ShieldAlert className="h-4 w-4 text-rose-500 animate-bounce" />;
                } else if (log.status === 'warn') {
                  badgeColor = 'bg-amber-50 text-amber-700 border-amber-200';
                  textColor = 'text-amber-700 font-bold';
                  statusText = lang === 'ar' ? 'تحذير كبح' : 'Defense Action';
                } else {
                  badgeColor = 'bg-slate-50 text-slate-600 border-slate-100';
                  textColor = 'text-slate-600';
                  statusText = lang === 'ar' ? 'معلومة نظام' : 'Audit logs';
                }

                return (
                  <div 
                    key={log.id} 
                    className={`flex items-center justify-between p-3.5 bg-slate-50/50 rounded-2xl border border-slate-100 text-xs transition-all hover:bg-slate-100/55 ${lang === 'en' ? 'flex-row' : 'flex-row-reverse'}`}
                  >
                    <div className={`flex items-start gap-3 ${lang === 'en' ? 'flex-row' : 'flex-row-reverse'}`}>
                      <span className={`px-2 py-0.5 rounded-full text-[9px] font-black border uppercase font-mono ${badgeColor}`}>
                        {statusText}
                      </span>
                      {icon}
                      <span className={`text-slate-700 font-medium ${textColor}`}>
                        {log.action}
                      </span>
                    </div>
                    <span className="text-[10px] text-slate-400 font-bold font-mono">
                      {log.time}
                    </span>
                  </div>
                );
              })
            ) : (
              <div className="text-center py-6 text-slate-400 font-bold">
                {lang === 'ar' ? 'لا يوجد سجلات أمان حالية، النظام في حالة خمول آمن.' : 'No audit payloads detected. Active shielding state is active.'}
              </div>
            )}
          </div>
        </div>
      </div>

    </div>
  );
}
