

import React, { useState, useCallback, useMemo, useEffect } from 'react';
import type { BalancingProcessData, Phase as PhaseEnum, RotorType, BalancingType, BalancingMethod, TestResult } from './types';
import { Phase } from './types';
import { getBalancingMethodSuggestion, getFailureAnalysis } from './services/geminiService';

// --- ICONS --- //
const CheckIcon: React.FC<{ className?: string }> = ({ className }) => (
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" className={className || "w-5 h-5"}><path fillRule="evenodd" d="M16.704 4.153a.75.75 0 01.143 1.052l-8 10.5a.75.75 0 01-1.127.075l-4.5-4.5a.75.75 0 011.06-1.06l3.894 3.893 7.48-9.817a.75.75 0 011.052-.143z" clipRule="evenodd" /></svg>
);
const SparklesIcon: React.FC<{ className?: string }> = ({ className }) => (
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" className={className || "w-5 h-5"}><path fillRule="evenodd" d="M10.868 2.884c-.321-.772-1.415-.772-1.736 0l-1.83 4.401-4.753.381c-.833.067-1.171 1.107-.536 1.651l3.62 3.102-1.106 4.637c-.194.813.691 1.456 1.405 1.02L10 15.591l4.069 2.485c.713.436 1.598-.207 1.404-1.02l-1.106-4.637 3.62-3.102c.635-.544.297-1.584-.536-1.65l-4.752-.382-1.831-4.401z" clipRule="evenodd" /></svg>
);
const ChevronRightIcon: React.FC<{ className?: string }> = ({ className }) => (
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" className={className || "w-5 h-5"}><path fillRule="evenodd" d="M7.21 14.77a.75.75 0 01.02-1.06L11.168 10 7.23 6.29a.75.75 0 111.04-1.08l4.5 4.25a.75.75 0 010 1.08l-4.5 4.25a.75.75 0 01-1.06-.02z" clipRule="evenodd" /></svg>
);
const InfoIcon: React.FC<{ className?: string }> = ({ className }) => (
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" className={className || "w-5 h-5"}><path fillRule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zM8.94 6.94a.75.75 0 11-1.061-1.061 3 3 0 112.871 5.026v.345a.75.75 0 01-1.5 0v-.5c0-.72.57-1.172 1.081-1.287A1.5 1.5 0 108.94 6.94zM10 15a1 1 0 100-2 1 1 0 000 2z" clipRule="evenodd" /></svg>
);
const SpinnerIcon: React.FC<{ className?: string }> = ({ className }) => (
    <svg className={`animate-spin ${className || "h-5 w-5"}`} xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24"><circle className="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="4"></circle><path className="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path></svg>
);
const PrintIcon: React.FC<{ className?: string }> = ({ className }) => (
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" className={className || "w-5 h-5"}><path fillRule="evenodd" d="M5 2.75C5 1.784 5.784 1 6.75 1h6.5c.966 0 1.75.784 1.75 1.75v3.552c.377.135.74.3.992.508l.51.425a1.875 1.875 0 010 2.53l-.51.425a1.875 1.875 0 01-.992.508v5.552A1.75 1.75 0 0113.25 19h-6.5A1.75 1.75 0 015 17.25V11.7c-.377-.135-.74-.3-.992-.508l-.51-.425a1.875 1.875 0 010-2.53l.51-.425a1.875 1.875 0 01.992-.508V2.75zM6.5 2.5a.25.25 0 00-.25.25v1c0 .138.112.25.25.25h6.5a.25.25 0 00.25-.25v-1a.25.25 0 00-.25-.25h-6.5zM6.5 11.25a.25.25 0 00-.25.25v4a.25.25 0 00.25.25h6.5a.25.25 0 00.25-.25v-4a.25.25 0 00-.25-.25h-6.5zM8.5 6.75a.75.75 0 00-1.5 0v.5h1.5v-.5z" clipRule="evenodd" /></svg>
);


// --- UI COMPONENTS --- //
const Card: React.FC<{ children: React.ReactNode; className?: string }> = ({ children, className }) => (
    <div className={`bg-slate-800/50 border border-slate-700 rounded-lg p-6 print-card ${className}`}>{children}</div>
);

const Button = React.forwardRef<HTMLButtonElement, React.ButtonHTMLAttributes<HTMLButtonElement> & { variant?: 'primary' | 'secondary' | 'ghost', leftIcon?: React.ReactNode, rightIcon?: React.ReactNode }>(
    ({ className, children, variant = 'primary', leftIcon, rightIcon, ...props }, ref) => {
        const baseClasses = "inline-flex items-center justify-center rounded-md px-4 py-2 text-sm font-semibold transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-offset-slate-900 disabled:opacity-50 disabled:pointer-events-none";
        const variantClasses = {
            primary: "bg-sky-600 text-white hover:bg-sky-700 focus:ring-sky-500",
            secondary: "bg-slate-600 text-slate-100 hover:bg-slate-700 focus:ring-slate-500",
            ghost: "hover:bg-slate-700/50 text-slate-300 focus:ring-slate-500"
        };
        return (
            <button className={`${baseClasses} ${variantClasses[variant]} ${className}`} {...props} ref={ref}>
                {leftIcon && <span className="mr-2 -ml-1">{leftIcon}</span>}
                {children}
                {rightIcon && <span className="ml-2 -mr-1">{rightIcon}</span>}
            </button>
        );
    }
);
Button.displayName = "Button";


interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
    label: string;
    helperText?: string;
    error?: string;
}
const Input: React.FC<InputProps> = ({ label, id, helperText, error, ...props }) => (
    <div>
        <label htmlFor={id} className="block text-sm font-medium text-slate-300 mb-1">{label}</label>
        <input id={id} className={`block w-full bg-slate-700/50 border rounded-md shadow-sm py-2 px-3 focus:outline-none focus:ring-sky-500 focus:border-sky-500 sm:text-sm read-only:bg-slate-800/60 ${error ? 'border-red-500' : 'border-slate-600'}`} {...props} />
        {!error && helperText && <p className="mt-1 text-xs text-slate-400">{helperText}</p>}
        {error && <p className="mt-1 text-xs text-red-400">{error}</p>}
    </div>
);

interface SelectProps extends React.SelectHTMLAttributes<HTMLSelectElement> {
    label: string;
    children: React.ReactNode;
    error?: string;
}
const Select: React.FC<SelectProps> = ({ label, id, children, error, ...props }) => (
    <div>
        <label htmlFor={id} className="block text-sm font-medium text-slate-300 mb-1">{label}</label>
        <select id={id} className={`block w-full bg-slate-700/50 border rounded-md shadow-sm py-2 px-3 focus:outline-none focus:ring-sky-500 focus:border-sky-500 sm:text-sm disabled:bg-slate-800/60 disabled:cursor-not-allowed ${error ? 'border-red-500' : 'border-slate-600'}`} {...props}>
            {children}
        </select>
         {error && <p className="mt-1 text-xs text-red-400">{error}</p>}
    </div>
);

interface CheckboxProps extends React.InputHTMLAttributes<HTMLInputElement> {
    label: string;
    error?: string;
}
const Checkbox: React.FC<CheckboxProps> = ({ label, id, error, ...props }) => (
    <div className="relative flex items-start">
        <div className="flex flex-col">
            <div className="flex h-6 items-center">
                <input id={id} type="checkbox" className="h-4 w-4 rounded border-slate-600 bg-slate-700 text-sky-600 focus:ring-sky-500" {...props} />
                <div className="ml-3 text-sm leading-6">
                    <label htmlFor={id} className="font-medium text-slate-300">{label}</label>
                </div>
            </div>
            {error && <p className="mt-1 text-xs text-red-400 ml-9">{error}</p>}
        </div>
    </div>
);

interface AlertProps {
    title: string;
    children: React.ReactNode;
    icon?: React.ReactElement<{ className?: string }>;
    type?: 'info' | 'ai';
}
const Alert: React.FC<AlertProps> = ({ title, children, icon, type = 'info' }) => {
    const typeClasses = {
        info: 'bg-sky-900/50 border-sky-700 text-sky-200',
        ai: 'bg-purple-900/50 border-purple-700 text-purple-200'
    };
    const iconColor = type === 'ai' ? 'text-purple-400' : 'text-sky-400';
    return (
        <div className={`rounded-md p-4 border ${typeClasses[type]}`}>
            <div className="flex">
                <div className="flex-shrink-0">
                    {icon && React.cloneElement(icon, { className: `h-5 w-5 ${iconColor}` })}
                </div>
                <div className="ml-3">
                    <h3 className="text-sm font-medium">{title}</h3>
                    <div className="mt-2 text-sm prose prose-sm prose-invert" dangerouslySetInnerHTML={{ __html: typeof children === 'string' ? children : '' }} />
                </div>
            </div>
        </div>
    );
};

// --- PHASE COMPONENTS --- //
type PhaseProps = {
    data: BalancingProcessData;
    setData: React.Dispatch<React.SetStateAction<BalancingProcessData>>;
    aiSuggestion: string;
    setAiSuggestion: React.Dispatch<React.SetStateAction<string>>;
    aiLoading: boolean;
    setAiLoading: React.Dispatch<React.SetStateAction<boolean>>;
    errors: Record<string, string>;
};

const Phase1Form: React.FC<PhaseProps> = ({ data, setData, errors }) => {
    const handleChange = (e: React.ChangeEvent<HTMLInputElement | HTMLTextAreaElement | HTMLSelectElement>) => {
        const { name, value, type } = e.target;
        const [section, key] = name.split('.');

        if (name === 'rotorData.model') {
            const newModel = value;
            let newBalanceGrade = '';
            switch (newModel) {
                case 'Pump':
                case 'Blower':
                    newBalanceGrade = 'G6.3';
                    break;
                case 'Turbine':
                    newBalanceGrade = 'G1';
                    break;
                case 'Compressor':
                    newBalanceGrade = 'G2.5';
                    break;
                case 'Other':
                default:
                    newBalanceGrade = '';
                    break;
            }
            setData(prev => ({
                ...prev,
                rotorData: {
                    ...prev.rotorData,
                    model: newModel,
                    balanceGrade: newBalanceGrade,
                }
            }));
            return;
        }

        const isCheckbox = type === 'checkbox';
        const checked = isCheckbox ? (e.target as HTMLInputElement).checked : undefined;
        
        setData(prev => ({
            ...prev,
            [section]: {
                // @ts-ignore
                ...prev[section],
                [key]: isCheckbox ? checked : value,
            }
        }));
    };

    return (
        <div className="space-y-8">
            <Card>
                <h3 className="text-lg font-semibold text-slate-100 mb-4">1. การรวบรวมข้อมูลโรเตอร์</h3>
                <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                    <Input label="รหัสเครื่องจักร" name="rotorData.machineId" value={data.rotorData.machineId} onChange={handleChange} required error={errors['rotorData.machineId']} />
                    <Select label="ประเภทเครื่องจักร" name="rotorData.model" value={data.rotorData.model} onChange={handleChange} required error={errors['rotorData.model']}>
                        <option value="">เลือกประเภท</option>
                        <option value="Pump">Pump</option>
                        <option value="Turbine">Turbine</option>
                        <option value="Compressor">Compressor</option>
                        <option value="Blower">Blower</option>
                        <option value="Other">อื่นๆ</option>
                    </Select>
                    <Input label="มวล (กก.)" name="rotorData.mass" value={data.rotorData.mass} onChange={handleChange} type="number" required error={errors['rotorData.mass']} />
                    <Input label="ความกว้างโรเตอร์ (มม.)" name="rotorData.rotorWidth" value={data.rotorData.rotorWidth} onChange={handleChange} type="number" required error={errors['rotorData.rotorWidth']} />
                    <Input label="เส้นผ่านศูนย์กลางโรเตอร์ (มม.)" name="rotorData.rotorDimensions" value={data.rotorData.rotorDimensions} onChange={handleChange} type="number" required error={errors['rotorData.rotorDimensions']} />
                    <Input label="ความเร็วใช้งานจริง (RPM)" name="rotorData.actualOperatingSpeed" value={data.rotorData.actualOperatingSpeed} onChange={handleChange} type="number" required error={errors['rotorData.actualOperatingSpeed']} />
                    <Input label="ความเร็ววิกฤต (RPM)" name="rotorData.criticalSpeed" value={data.rotorData.criticalSpeed} onChange={handleChange} type="number" required error={errors['rotorData.criticalSpeed']} />
                    <Select label="ประเภทโรเตอร์" name="rotorData.rotorType" value={data.rotorData.rotorType} onChange={handleChange} disabled>
                        <option value="">เลือกอัตโนมัติ</option>
                        <option value="Rigid">Rigid</option>
                        <option value="Flexible">Flexible</option>
                    </Select>
                    <Input 
                        label="Balance Quality Grade (เช่น G6.3, G2.5)" 
                        name="rotorData.balanceGrade" 
                        value={data.rotorData.balanceGrade}
                        onChange={handleChange}
                        helperText={data.rotorData.model === 'Other' ? "ระบุเกรดคุณภาพตามมาตรฐาน ISO 21940-11" : "กำหนดอัตโนมัติตามประเภทเครื่องจักรและมาตรฐาน ISO 21940-11"}
                        required 
                        error={errors['rotorData.balanceGrade']}
                        readOnly={data.rotorData.model !== 'Other'}
                        placeholder={data.rotorData.model === 'Other' ? "ระบุเกรดด้วยตนเอง (เช่น G6.3)" : "เลือกประเภทเครื่องจักรก่อน"}
                    />
                    <div className="md:col-span-2">
                         <label htmlFor="rotorData.repairHistory" className="block text-sm font-medium text-slate-300 mb-1">ประวัติการซ่อม</label>
                         <textarea id="rotorData.repairHistory" name="rotorData.repairHistory" value={data.rotorData.repairHistory} onChange={handleChange} rows={3} className="block w-full bg-slate-700/50 border border-slate-600 rounded-md shadow-sm py-2 px-3 focus:outline-none focus:ring-sky-500 focus:border-sky-500 sm:text-sm" />
                    </div>
                </div>
            </Card>
            <Card>
                <h3 className="text-lg font-semibold text-slate-100 mb-4">2. การตรวจสอบทางกายภาพ</h3>
                <div className="space-y-4">
                    <Checkbox label="ทำความสะอาดและปราศจากสนิม, สีหลุดร่อน, หรือสิ่งปนเปื้อนอื่นๆ" name="physicalInspection.isCleaned" checked={data.physicalInspection.isCleaned} onChange={handleChange} />
                    <Input label="วิธีการตรวจสอบรอยร้าว (เช่น NDT, MPI)" name="physicalInspection.crackCheckMethod" value={data.physicalInspection.crackCheckMethod} onChange={handleChange} />
                    <Checkbox label="พบความเสียหายหรือข้อบกพร่องบนชิ้นส่วน" name="physicalInspection.hasDamage" checked={data.physicalInspection.hasDamage} onChange={handleChange} />
                    {data.physicalInspection.hasDamage && (
                        <Card className="border-amber-500/50 bg-amber-900/20">
                            <h4 className="font-semibold text-amber-300 mb-3">การดำเนินการซ่อมแซม</h4>
                            <div className="space-y-4">
                                <Checkbox label="การซ่อมแซมได้รับการอนุมัติจากวิศวกร" name="physicalInspection.supervisorApprovedRepair" checked={data.physicalInspection.supervisorApprovedRepair} onChange={handleChange} />
                                <Checkbox label="ตรวจสอบชิ้นส่วนซ้ำหลังการซ่อมแซม" name="physicalInspection.reinspected" checked={data.physicalInspection.reinspected} onChange={handleChange} />
                            </div>
                        </Card>
                    )}
                </div>
            </Card>
        </div>
    );
};

const Phase2Form: React.FC<PhaseProps> = ({ data, setData, aiSuggestion, setAiSuggestion, aiLoading, setAiLoading, errors }) => {
    const handleChange = (e: React.ChangeEvent<HTMLInputElement | HTMLSelectElement>) => {
        const { name, value, type } = e.target;
        const [section, key] = name.split('.');
        const isCheckbox = type === 'checkbox';
        const checked = isCheckbox ? (e.target as HTMLInputElement).checked : undefined;
        
        setData(prev => ({
            ...prev,
            [section]: {
                // @ts-ignore
                ...prev[section],
                [key]: isCheckbox ? checked : value,
            }
        }));
    };

    const handleGetSuggestion = useCallback(async () => {
        setAiLoading(true);
        setAiSuggestion('');
        const suggestion = await getBalancingMethodSuggestion(data);
        setAiSuggestion(suggestion);
        setAiLoading(false);
    }, [data, setAiLoading, setAiSuggestion]);

    const uPer = useMemo(() => parseFloat(data.balancingPlan.permissibleResidualUnbalance), [data.balancingPlan.permissibleResidualUnbalance]);
    const uQmar = useMemo(() => parseFloat(data.balancingPlan.uqmarValue), [data.balancingPlan.uqmarValue]);
    
    const assessmentResult = useMemo(() => {
        if (isNaN(uPer) || isNaN(uQmar) || uQmar <= 0) {
            return {
                ratio: NaN,
                text: 'รอการป้อนข้อมูล...',
                status: 'neutral',
                colorClasses: 'bg-slate-700 text-slate-300 border border-slate-600'
            };
        }
        const ratio = uPer / uQmar;
        if (ratio > 10) {
            return {
                ratio: ratio,
                text: 'ยอดเยี่ยม',
                status: 'excellent',
                colorClasses: 'bg-green-900/60 text-green-300 border border-green-700/50'
            };
        } else if (ratio >= 3) {
            return {
                ratio: ratio,
                text: 'ยอมรับได้',
                status: 'acceptable',
                colorClasses: 'bg-sky-900/60 text-sky-300 border border-sky-700/50'
            };
        } else {
            return {
                ratio: ratio,
                text: 'มีความเสี่ยง / ไม่แนะนำ',
                status: 'risky',
                colorClasses: 'bg-red-900/60 text-red-300 border border-red-700/50'
            };
        }
    }, [uPer, uQmar]);

    const AssessmentItem: React.FC<{label: string, value: React.ReactNode, status?: 'ok' | 'bad' | 'neutral'}> = ({label, value, status = 'neutral'}) => {
        const statusColor = {
            ok: 'text-green-400',
            bad: 'text-red-400',
            neutral: 'text-slate-200'
        };
        return (
            <div className="flex justify-between items-center py-1">
                <span className="text-sm text-slate-400">{label}</span>
                <span className={`text-sm font-semibold ${statusColor[status]}`}>{value}</span>
            </div>
        );
    };


    return (
        <div className="space-y-8">
            <Card>
                <h3 className="text-lg font-semibold text-slate-100 mb-4">3. การเลือกประเภทการถ่วงสมดุล</h3>
                <div className="flex flex-col sm:flex-row gap-4">
                    <div className="flex-1 border border-slate-700 rounded-lg p-4 bg-slate-800">
                        <h4 className="font-bold text-slate-200">การถ่วงสมดุลเฉพาะชิ้นส่วน</h4>
                        <p className="text-sm text-slate-400 mt-1">สำหรับงานทั่วไป (เช่น G6.3) หรือเมื่อการประกอบไม่มีปัญหา</p>
                    </div>
                    <div className="flex-1 border border-slate-700 rounded-lg p-4 bg-slate-800">
                        <h4 className="font-bold text-slate-200">การถ่วงสมดุลทั้งชุดประกอบ</h4>
                        <p className="text-sm text-slate-400 mt-1">สำหรับงานความเร็วสูง/ความแม่นยำสูง (&lt;G2.5) หรือเพื่อแก้ไขปัญหาที่การถ่วงสมดุลชิ้นส่วนแก้ไม่ได้</p>
                    </div>
                </div>
                <div className="mt-6 grid grid-cols-1 md:grid-cols-2 gap-6 items-end">
                    <Select label="ประเภทการถ่วงสมดุลที่เลือก" name="balancingPlan.balancingType" value={data.balancingPlan.balancingType} onChange={handleChange} required error={errors['balancingPlan.balancingType']}>
                        <option value="">เลือกประเภท</option>
                        <option value="Component">การถ่วงสมดุลเฉพาะชิ้นส่วน</option>
                        <option value="Assembly">การถ่วงสมดุลทั้งชุดประกอบ</option>
                    </Select>
                    <Input label="มาตรฐานลิ่ม" name="balancingPlan.keyStandard" value={data.balancingPlan.keyStandard} readOnly placeholder="เลือกอัตโนมัติ" />
                </div>
            </Card>
            <Card>
                <h3 className="text-lg font-semibold text-slate-100 mb-4">4. การเลือกวิธีการถ่วงสมดุล</h3>
                <div className="grid grid-cols-1 md:grid-cols-3 gap-6 mb-6">
                    <Alert title="Static Balance" icon={<InfoIcon />}>สำหรับโรเตอร์แบบ Rigid ที่บาง (L/D ≤ 5)</Alert>
                    <Alert title="Dynamic Balance" icon={<InfoIcon />}>สำหรับโรเตอร์แบบ Rigid ที่ยาว (L/D > 5)</Alert>
                    <Alert title="Field Balance" icon={<InfoIcon />}>สำหรับโรเตอร์แบบ Flexible หรือการถ่วงสมดุลเครื่องจักรที่ประกอบแล้ว ณ สถานที่ติดตั้ง</Alert>
                </div>
                <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
                    <Input label="L/D" name="balancingPlan.ldRatio" value={data.balancingPlan.ldRatio} readOnly placeholder="คำนวณอัตโนมัติ" />
                     <Input
                        label="วิธีการถ่วงสมดุล"
                        name="balancingPlan.selectedMethod"
                        value={data.balancingPlan.selectedMethod}
                        readOnly
                        placeholder="เลือกอัตโนมัติ"
                        helperText="พิจารณาจากประเภทโรเตอร์และ L/D"
                    />
                    <Input 
                        label="ค่าความไม่สมดุลที่ยอมรับได้ (g·mm)" 
                        name="balancingPlan.permissibleResidualUnbalance"
                        value={data.balancingPlan.permissibleResidualUnbalance} 
                        readOnly 
                        placeholder="คำนวณอัตโนมัติ" 
                        helperText="จาก G, มวล, ความเร็ว"
                    />
                </div>
                <div className="mt-6">
                     <Button variant="secondary" onClick={handleGetSuggestion} disabled={aiLoading} leftIcon={aiLoading ? <SpinnerIcon /> : <SparklesIcon />}>
                        {aiLoading ? 'กำลังรับคำแนะนำ...' : 'รับคำแนะนำจาก AI'}
                    </Button>
                </div>
                {aiSuggestion && (
                    <div className="mt-6">
                        <Alert title="คำแนะนำวิธีการถ่วงสมดุลจาก AI" type="ai" icon={<SparklesIcon />}>
                           {aiSuggestion}
                        </Alert>
                    </div>
                )}
            </Card>
            <Card>
                <h3 className="text-lg font-semibold text-slate-100 mb-4">5. การเลือกเครื่องถ่วงสมดุล</h3>
                <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                    <Input label="ใบรับรองเครื่องจักร (เช่น ISO 21940-14)" name="balancingPlan.machineCert" value={data.balancingPlan.machineCert} onChange={handleChange} />
                     <Input 
                        label="ค่า Uqmar (ความเที่ยงตรงของเครื่องจักร)"
                        name="balancingPlan.uqmarValue"
                        value={data.balancingPlan.uqmarValue} 
                        onChange={handleChange}
                        type="number"
                        helperText="ป้อนค่าความเที่ยงตรงของเครื่องจักร (g·mm) จากใบรับรอง"
                        required
                        error={errors['balancingPlan.uqmarValue']}
                    />
                    <Input 
                        label="ศูนย์บริการถ่วงสมดุล"
                        name="balancingPlan.vendorBalancingWorkshop"
                        value={data.balancingPlan.vendorBalancingWorkshop}
                        readOnly
                        placeholder="เลือกอัตโนมัติ"
                        helperText="Vendor A สำหรับขนาด > 1000 มม., นอกนั้นเป็น Vendor B"
                    />
                    <div className="md:col-span-2">
                        <Checkbox label="เครื่องจักรสามารถรองรับน้ำหนัก, ขนาด, และขนาดเพลาได้" name="balancingPlan.machineCapacityOk" checked={data.balancingPlan.machineCapacityOk} onChange={handleChange} required error={errors['balancingPlan.machineCapacityOk']}/>
                    </div>
                </div>

                <div className="mt-6 border-t border-slate-700 pt-4">
                    <h4 className="text-md font-semibold text-slate-200 mb-2">การประเมินความสามารถของเครื่องจักร</h4>
                    
                    <div className="bg-slate-800 rounded-lg p-4 mt-2 space-y-1 border border-slate-700">
                        <AssessmentItem 
                            label="ค่าความไม่สมดุลคงเหลือที่ยอมรับได้ (U_per)"
                            value={!isNaN(uPer) ? `${uPer.toFixed(2)} g·mm` : 'ยังไม่ได้คำนวณ'}
                        />
                         <AssessmentItem
                            label="ค่าความเที่ยงตรงจริงของเครื่องจักร (Uqmar)"
                            value={!isNaN(uQmar) ? `${uQmar.toFixed(2)} g·mm` : 'ยังไม่ได้ป้อน'}
                        />
                        <AssessmentItem
                            label="อัตราส่วนการประเมิน (U_per / Uqmar)"
                            value={!isNaN(assessmentResult.ratio) ? assessmentResult.ratio.toFixed(2) : 'N/A'}
                            status={
                                assessmentResult.status === 'excellent' || assessmentResult.status === 'acceptable' ? 'ok' :
                                assessmentResult.status === 'risky' ? 'bad' : 'neutral'
                            }
                        />
                    </div>
    
                    <div className={`mt-3 text-center text-sm font-medium p-2 rounded-md border ${assessmentResult.colorClasses}`}>
                        สถานะ: {assessmentResult.text}
                    </div>
                </div>

            </Card>
        </div>
    );
};

const Phase4Form: React.FC<PhaseProps> = ({ data, setData, aiSuggestion, setAiSuggestion, aiLoading, setAiLoading, errors }) => {
    const handleChange = (e: React.ChangeEvent<HTMLInputElement | HTMLSelectElement>) => {
        const { name, value, type } = e.target;
        const [section, key] = name.split('.');
        const isCheckbox = type === 'checkbox';
        const checked = isCheckbox ? (e.target as HTMLInputElement).checked : undefined;
        
        setData(prev => ({
            ...prev,
            [section]: {
                // @ts-ignore
                ...prev[section],
                [key]: isCheckbox ? checked : value,
            }
        }));
    };
    
    const handleGetAnalysis = useCallback(async () => {
        setAiLoading(true);
        setAiSuggestion(''); // Re-using aiSuggestion state for analysis result
        const analysis = await getFailureAnalysis(data);
        setAiSuggestion(analysis);
        setAiLoading(false);
    }, [data, setAiLoading, setAiSuggestion]);

    return (
        <div className="space-y-8">
            <Card>
                <h3 className="text-lg font-semibold text-slate-100 mb-4">6.1 การตรวจสอบเอกสาร (รายงานการถ่วงสมดุล)</h3>
                <div className="space-y-4">
                    <Checkbox label="รายงานครบถ้วนและถูกต้อง (ข้อมูลเครื่องจักร, ผู้ปฏิบัติงาน, วันที่)" name="verificationData.reportComplete" checked={data.verificationData.reportComplete} onChange={handleChange} />
                    <Checkbox label="ระบุเกรดคุณภาพการถ่วงสมดุลและค่าความไม่สมดุลที่ยอมรับได้ (U_per) ถูกต้อง" name="verificationData.gradeSpecified" checked={data.verificationData.gradeSpecified} onChange={handleChange} />
                    <Checkbox label="รายงานแสดงค่าความไม่สมดุล ก่อน และ หลัง การแก้ไข" name="verificationData.unbalanceRecorded" checked={data.verificationData.unbalanceRecorded} onChange={handleChange} />
                    <Checkbox label="บันทึกข้อมูลเครื่องถ่วงสมดุลและวันที่สอบเทียบล่าสุด" name="verificationData.toolsCalibrated" checked={data.verificationData.toolsCalibrated} onChange={handleChange} />
                </div>
            </Card>
            <Card>
                <h3 className="text-lg font-semibold text-slate-100 mb-4">6.2 การตรวจสอบหลังการถ่วงสมดุล</h3>
                <div className="space-y-4">
                    <Checkbox label="ไม่เกิดความเสียหายใหม่ระหว่างการถ่วงสมดุล (เช่น รอยบิ่น, รอยบุบ, หรือรอยร้าวบนใบพัด, ครีบ ฯลฯ)" name="verificationData.noNewDamage" checked={data.verificationData.noNewDamage} onChange={handleChange} />
                    <Checkbox label="การเพิ่ม/ลด น้ำหนัก มีความมั่นคงและผ่านการตรวจสอบ (เช่น PT)" name="verificationData.correctionSecure" checked={data.verificationData.correctionSecure} onChange={handleChange} />
                </div>
            </Card>
            <Card>
                <h3 className="text-lg font-semibold text-slate-100 mb-4">6.3 การยอมรับค่าความไม่สมดุลคงเหลือ</h3>
                <fieldset>
                    <legend className="text-sm font-medium text-slate-300 mb-2">ผลลัพธ์สุดท้าย</legend>
                    <div className="flex gap-x-6">
                        <div className="flex items-center gap-x-2"><input id="pass" name="verificationData.finalResult" type="radio" value="Pass" checked={data.verificationData.finalResult === 'Pass'} onChange={handleChange} className="h-4 w-4 border-slate-600 bg-slate-700 text-sky-600 focus:ring-sky-500" /><label htmlFor="pass">ผ่าน</label></div>
                        <div className="flex items-center gap-x-2"><input id="fail" name="verificationData.finalResult" type="radio" value="Fail" checked={data.verificationData.finalResult === 'Fail'} onChange={handleChange} className="h-4 w-4 border-slate-600 bg-slate-700 text-sky-600 focus:ring-sky-500" /><label htmlFor="fail">ไม่ผ่าน</label></div>
                    </div>
                     {errors['verificationData.finalResult'] && <p className="mt-2 text-xs text-red-400">{errors['verificationData.finalResult']}</p>}
                </fieldset>
                {data.verificationData.finalResult === 'Fail' && (
                    <div className="mt-6">
                        <Button variant="secondary" onClick={handleGetAnalysis} disabled={aiLoading} leftIcon={aiLoading ? <SpinnerIcon /> : <SparklesIcon />}>
                            {aiLoading ? 'กำลังวิเคราะห์...' : 'รับการวิเคราะห์ความล้มเหลวจาก AI'}
                        </Button>
                    </div>
                )}
                 {data.verificationData.finalResult === 'Fail' && aiSuggestion && (
                    <div className="mt-6">
                        <Alert title="การวิเคราะห์ความล้มเหลวจาก AI" type="ai" icon={<SparklesIcon />}>
                           {aiSuggestion}
                        </Alert>
                    </div>
                )}
            </Card>
        </div>
    );
};

const SummaryScreen: React.FC<{data: BalancingProcessData}> = ({ data }) => {
    const SummaryItem: React.FC<{label: string, value: React.ReactNode}> = ({label, value}) => (
        <div>
            <dt className="text-sm font-medium text-slate-400">{label}</dt>
            <dd className="mt-1 text-sm text-slate-200">{value || 'N/A'}</dd>
        </div>
    );
    
    return (
        <Card>
            <h3 className="text-xl font-bold text-sky-400 mb-6">รายงานสรุปกระบวนการถ่วงสมดุล</h3>
            <div className="space-y-6">
                <section>
                    <h4 className="font-semibold text-lg text-slate-100 border-b border-slate-700 pb-2 mb-4">ข้อมูลโรเตอร์และการตรวจสอบ</h4>
                    <dl className="grid grid-cols-2 md:grid-cols-4 gap-x-4 gap-y-6">
                        <SummaryItem label="รหัสเครื่องจักร" value={data.rotorData.machineId} />
                        <SummaryItem label="ประเภทเครื่องจักร" value={data.rotorData.model} />
                        <SummaryItem label="มวล" value={data.rotorData.mass ? `${data.rotorData.mass} kg` : 'N/A'} />
                        <SummaryItem label="ความกว้างโรเตอร์" value={data.rotorData.rotorWidth ? `${data.rotorData.rotorWidth} mm` : 'N/A'} />
                        <SummaryItem label="เส้นผ่านศูนย์กลางโรเตอร์" value={data.rotorData.rotorDimensions ? `${data.rotorData.rotorDimensions} mm` : 'N/A'} />
                        <SummaryItem label="ประเภทโรเตอร์" value={data.rotorData.rotorType} />
                        <SummaryItem label="ความเร็วใช้งาน" value={data.rotorData.actualOperatingSpeed ? `${data.rotorData.actualOperatingSpeed} RPM` : 'N/A'} />
                        <SummaryItem label="ความเร็ววิกฤต" value={data.rotorData.criticalSpeed ? `${data.rotorData.criticalSpeed} RPM` : 'N/A'} />
                        <SummaryItem label="เกรดความสมดุล" value={data.rotorData.balanceGrade} />
                        <SummaryItem label="พบความเสียหาย?" value={data.physicalInspection.hasDamage ? 'ใช่' : 'ไม่'} />
                        <SummaryItem label="วิธีการตรวจสอบรอยร้าว" value={data.physicalInspection.crackCheckMethod} />
                    </dl>
                    <div className="mt-6">
                        <dt className="text-sm font-medium text-slate-400">ประวัติการซ่อม</dt>
                        <dd className="mt-1 text-sm text-slate-200 whitespace-pre-wrap font-sans">{data.rotorData.repairHistory || 'ไม่มีข้อมูล'}</dd>
                    </div>
                </section>
                <section>
                    <h4 className="font-semibold text-lg text-slate-100 border-b border-slate-700 pb-2 mb-4">แผนการถ่วงสมดุล</h4>
                     <dl className="grid grid-cols-2 md:grid-cols-4 gap-x-4 gap-y-6">
                        <SummaryItem label="ประเภทการถ่วงสมดุล" value={
                            data.balancingPlan.balancingType === 'Component' ? 'การถ่วงสมดุลเฉพาะชิ้นส่วน' :
                            data.balancingPlan.balancingType === 'Assembly' ? 'การถ่วงสมดุลทั้งชุดประกอบ' : 'N/A'
                        } />
                        <SummaryItem label="มาตรฐานลิ่ม" value={data.balancingPlan.keyStandard} />
                        <SummaryItem label="L/D" value={data.balancingPlan.ldRatio} />
                        <SummaryItem label="วิธีการถ่วงสมดุล" value={data.balancingPlan.selectedMethod} />
                        <SummaryItem label="ค่าความไม่สมดุลที่ยอมรับได้" value={data.balancingPlan.permissibleResidualUnbalance ? `${data.balancingPlan.permissibleResidualUnbalance} g·mm` : 'N/A'} />
                        <SummaryItem label="ศูนย์บริการถ่วงสมดุล" value={data.balancingPlan.vendorBalancingWorkshop} />
                    </dl>
                </section>
                 <section>
                    <h4 className="font-semibold text-lg text-slate-100 border-b border-slate-700 pb-2 mb-4">การตรวจสอบ</h4>
                     <dl className="grid grid-cols-2 md:grid-cols-4 gap-x-4 gap-y-6">
                        <SummaryItem label="ผลลัพธ์สุดท้าย" value={<span className={data.verificationData.finalResult === 'Pass' ? 'text-green-400 font-bold' : 'text-red-400 font-bold'}>{data.verificationData.finalResult === 'Pass' ? 'ผ่าน' : data.verificationData.finalResult === 'Fail' ? 'ไม่ผ่าน' : 'N/A'}</span>} />
                    </dl>
                </section>
            </div>
        </Card>
    );
};


const Stepper: React.FC<{ phases: string[], currentPhaseIndex: number }> = ({ phases, currentPhaseIndex }) => (
    <nav aria-label="Progress" className="no-print">
        <ol role="list" className="flex items-center">
            {phases.map((phase, phaseIdx) => (
                <li key={phase} className={`relative ${phaseIdx !== phases.length - 1 ? 'pr-8 sm:pr-20' : ''}`}>
                    {phaseIdx < currentPhaseIndex ? (
                         <>
                            {/* Completed Step */}
                            <div className="absolute inset-0 flex items-center" aria-hidden="true">
                                <div className="h-0.5 w-full bg-sky-600" />
                            </div>
                            <div className="relative flex h-8 w-8 items-center justify-center rounded-full bg-sky-600">
                                <CheckIcon className="h-5 w-5 text-white" />
                            </div>
                            <span className="absolute -bottom-6 w-max text-xs text-slate-300">{phase.split(':')[0]}</span>
                         </>
                    ) : phaseIdx === currentPhaseIndex ? (
                        <>
                            {/* Current Step */}
                            <div className="absolute inset-0 flex items-center" aria-hidden="true">
                                <div className="h-0.5 w-full bg-slate-700" />
                            </div>
                            <div className="relative flex h-8 w-8 items-center justify-center rounded-full border-2 border-sky-600 bg-slate-800" aria-current="step">
                                <span className="h-2.5 w-2.5 rounded-full bg-sky-600" />
                            </div>
                             <span className="absolute -bottom-6 w-max text-xs font-semibold text-sky-400">{phase.split(':')[0]}</span>
                        </>
                    ) : (
                        <>
                            {/* Upcoming Step */}
                            <div className="absolute inset-0 flex items-center" aria-hidden="true">
                                <div className="h-0.5 w-full bg-slate-700" />
                            </div>
                            <div className="relative flex h-8 w-8 items-center justify-center rounded-full border-2 border-slate-700 bg-slate-800">
                                <span className="h-2.5 w-2.5 rounded-full bg-slate-700" />
                            </div>
                             <span className="absolute -bottom-6 w-max text-xs text-slate-500">{phase.split(':')[0]}</span>
                        </>
                    )}
                </li>
            ))}
        </ol>
    </nav>
);

const initialData: BalancingProcessData = {
    rotorData: {
        machineId: '', model: '', mass: '', rotorWidth: '', rotorDimensions: '',
        actualOperatingSpeed: '', criticalSpeed: '', rotorType: '', balanceGrade: '', repairHistory: ''
    },
    physicalInspection: {
        isCleaned: false, crackCheckMethod: '', hasDamage: false,
        supervisorApprovedRepair: false, reinspected: false
    },
    balancingPlan: {
        balancingType: '', keyStandard: '', machineCert: '', uqmarValue: '',
        machineCapacityOk: false, ldRatio: '', selectedMethod: '', aiSuggestion: '',
        permissibleResidualUnbalance: '', vendorBalancingWorkshop: ''
    },
    verificationData: {
        reportComplete: false, gradeSpecified: false, unbalanceRecorded: false,
        toolsCalibrated: false, noNewDamage: false, correctionSecure: false,
        finalResult: '', aiFailureAnalysis: ''
    }
};

const PHASES = Object.values(Phase);

export default function App() {
    // State management
    const [phase, setPhase] = useState<PhaseEnum>(Phase.PHASE_1);
    const [data, setData] = useState<BalancingProcessData>(initialData);
    const [errors, setErrors] = useState<Record<string, string>>({});
    const [aiSuggestion, setAiSuggestion] = useState('');
    const [aiLoading, setAiLoading] = useState(false);

    // Automatic calculations and derived state
    useEffect(() => {
        const { rotorWidth, rotorDimensions, actualOperatingSpeed, criticalSpeed, mass, balanceGrade } = data.rotorData;
        const width = parseFloat(rotorWidth);
        const diameter = parseFloat(rotorDimensions);
        const operatingSpeed = parseFloat(actualOperatingSpeed);
        const critSpeed = parseFloat(criticalSpeed);
        const rotorMass = parseFloat(mass);

        // L/D Ratio
        const ldRatio = (width && diameter) ? (width / diameter).toFixed(2) : '';

        // Rotor Type
        let rotorType: RotorType = '';
        if (critSpeed && operatingSpeed) {
            rotorType = (operatingSpeed / critSpeed) >= 0.75 ? 'Flexible' : 'Rigid';
        }
        
        // Permissible Residual Unbalance (U_per)
        // Formula: U_per = (9550 * G * W) / N
        let uPer = '';
        if (balanceGrade && rotorMass && operatingSpeed) {
            const gValue = parseFloat(balanceGrade.replace('G', ''));
            if (!isNaN(gValue)) {
                uPer = ((9550 * gValue * rotorMass) / operatingSpeed).toFixed(2);
            }
        }

        // Key Standard
        let keyStandard = 'ไม่จำเป็นต้องใช้มาตรฐานลิ่ม';
        if (data.balancingPlan.balancingType === 'Assembly') {
             keyStandard = 'ถ่วงสมดุลโดยไม่มีลิ่ม';
        } else if (data.balancingPlan.balancingType === 'Component') {
             keyStandard = 'ใช้ลิ่มครึ่งตัว (Half-key)';
        }

        // Vendor Workshop
        let vendor = '';
        if (diameter) {
            vendor = diameter > 1000 ? 'Vendor A' : 'Vendor B';
        }

        // Automatic Balancing Method Selection
        let selectedMethod: BalancingMethod = '';
        const ldNum = parseFloat(ldRatio);

        if (rotorType === 'Flexible') {
            selectedMethod = 'Field';
        } else if (rotorType === 'Rigid') {
            if (!isNaN(ldNum)) {
                if (ldNum <= 5) {
                    selectedMethod = 'Static';
                } else {
                    selectedMethod = 'Dynamic';
                }
            }
        }

        setData(prev => ({
            ...prev,
            rotorData: {
                ...prev.rotorData,
                rotorType: rotorType
            },
            balancingPlan: {
                ...prev.balancingPlan,
                ldRatio: ldRatio,
                permissibleResidualUnbalance: uPer,
                keyStandard: keyStandard,
                vendorBalancingWorkshop: vendor,
                selectedMethod: selectedMethod
            }
        }));
    }, [
        data.rotorData.rotorWidth, data.rotorData.rotorDimensions, data.rotorData.actualOperatingSpeed,
        data.rotorData.criticalSpeed, data.rotorData.mass, data.rotorData.balanceGrade, data.balancingPlan.balancingType
    ]);

    // Reset AI suggestion when phase changes
    useEffect(() => {
        setAiSuggestion('');
    }, [phase]);
    
    // Validation logic
    const validateCurrentPhase = useCallback((): boolean => {
        const newErrors: Record<string, string> = {};
        const requiredMsg = "จำเป็นต้องระบุ";

        switch (phase) {
            case Phase.PHASE_1:
                const { rotorData } = data;
                if (!rotorData.machineId) newErrors['rotorData.machineId'] = requiredMsg;
                if (!rotorData.model) newErrors['rotorData.model'] = requiredMsg;
                if (!rotorData.mass) newErrors['rotorData.mass'] = requiredMsg;
                if (!rotorData.rotorWidth) newErrors['rotorData.rotorWidth'] = requiredMsg;
                if (!rotorData.rotorDimensions) newErrors['rotorData.rotorDimensions'] = requiredMsg;
                if (!rotorData.actualOperatingSpeed) newErrors['rotorData.actualOperatingSpeed'] = requiredMsg;
                if (!rotorData.criticalSpeed) newErrors['rotorData.criticalSpeed'] = requiredMsg;
                if (!rotorData.balanceGrade) newErrors['rotorData.balanceGrade'] = requiredMsg;
                break;
            case Phase.PHASE_2:
                const { balancingPlan } = data;
                if (!balancingPlan.balancingType) newErrors['balancingPlan.balancingType'] = requiredMsg;
                if (!balancingPlan.uqmarValue) newErrors['balancingPlan.uqmarValue'] = requiredMsg;
                if (!balancingPlan.machineCapacityOk) newErrors['balancingPlan.machineCapacityOk'] = "จำเป็นต้องยืนยัน";
                break;
            case Phase.PHASE_4:
                 const { verificationData } = data;
                 if (!verificationData.finalResult) newErrors['verificationData.finalResult'] = requiredMsg;
                break;
        }

        setErrors(newErrors);
        return Object.keys(newErrors).length === 0;
    }, [data, phase]);
    
    // Navigation handlers
    const handleNext = () => {
        if (validateCurrentPhase()) {
            const currentPhaseIndex = PHASES.indexOf(phase);
            if (currentPhaseIndex < PHASES.length - 1) {
                setPhase(PHASES[currentPhaseIndex + 1]);
            }
        }
    };
    
    const handleBack = () => {
        const currentPhaseIndex = PHASES.indexOf(phase);
        if (currentPhaseIndex > 0) {
            setPhase(PHASES[currentPhaseIndex - 1]);
        }
    };
    
    const handleReset = () => {
        setData(initialData);
        setPhase(Phase.PHASE_1);
        setErrors({});
        setAiSuggestion('');
    };

    const handlePrint = () => window.print();

    // Phase rendering
    const renderPhase = () => {
        const props = { data, setData, aiSuggestion, setAiSuggestion, aiLoading, setAiLoading, errors };
        switch (phase) {
            case Phase.PHASE_1:
                return <Phase1Form {...props} />;
            case Phase.PHASE_2:
                return <Phase2Form {...props} />;
            case Phase.PHASE_4:
                return <Phase4Form {...props} />;
            case Phase.SUMMARY:
                return <SummaryScreen data={data} />;
            default:
                return <div>Invalid Phase</div>;
        }
    };

    const currentPhaseIndex = PHASES.indexOf(phase);

    return (
        <div className="max-w-5xl mx-auto px-4 sm:px-6 lg:px-8 py-12 print-container">
            <header className="text-center mb-12 no-print">
                <h1 className="text-3xl font-bold tracking-tight text-slate-100 sm:text-4xl">UBE Balancing Manual Standard</h1>
            </header>

            <div className="mb-16 no-print">
                <Stepper phases={PHASES} currentPhaseIndex={currentPhaseIndex} />
            </div>

            <main>
                <div className="mb-12">
                    <h2 className="text-2xl font-bold text-sky-400 mb-2 no-print">{PHASES[currentPhaseIndex]}</h2>
                </div>
                {renderPhase()}
            </main>

            <footer className="mt-16 flex justify-between items-center no-print">
                <div>
                    {currentPhaseIndex > 0 && phase !== Phase.SUMMARY && (
                        <Button variant="secondary" onClick={handleBack}>
                            ย้อนกลับ
                        </Button>
                    )}
                </div>
                <div className="flex gap-4">
                    {phase === Phase.SUMMARY ? (
                        <>
                           <Button variant="secondary" onClick={handlePrint} leftIcon={<PrintIcon />}>พิมพ์รายงาน</Button>
                           <Button variant="primary" onClick={handleReset}>เริ่มต้นใหม่</Button>
                        </>
                    ) : (
                        <Button variant="primary" onClick={handleNext} rightIcon={<ChevronRightIcon />}>
                           {phase === Phase.PHASE_4 ? 'ดูสรุป' : 'ถัดไป'}
                        </Button>
                    )}
                </div>
            </footer>
        </div>
    );
}
