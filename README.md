/* ==========================================================================
   STATE MANAGEMENT & DATA MODELS
   ========================================================================== */
let state = {
    staff: [],
    cases: [],
    rentals: [],
    sales: []
};

// Temp memory for closed sale stakeholders during form entry
let tempSaleStakeholders = [];

// Storage Keys
const STORAGE_KEY = 'broker_force_state_v1';

// Load state from Cloudflare D1 database (with localStorage fallback)
async function loadState() {
    try {
        const res = await fetch('/api/state');
        if (res.ok) {
            const cloudData = await res.json();
            state = {
                staff: cloudData.staff || [],
                cases: cloudData.cases || [],
                rentals: cloudData.rentals || [],
                sales: cloudData.sales || []
            };
            // Cache locally as backup
            localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
            console.log("Sync: Successfully loaded state from Cloudflare D1 Database.");
            return;
        }
    } catch (e) {
        console.warn("Sync: Cloudflare API unavailable, falling back to localStorage cache.", e);
    }

    // Fallback loading from local cache
    try {
        const data = localStorage.getItem(STORAGE_KEY);
        if (data) {
            state = JSON.parse(data);
            state.staff = state.staff || [];
            state.cases = state.cases || [];
            state.rentals = state.rentals || [];
            state.sales = state.sales || [];
        } else {
            initializeDemoData();
        }
    } catch (e) {
        console.error("Error reading localStorage", e);
        initializeDemoData();
    }
}

// Save state to localStorage as a local backup cache
function saveState() {
    try {
        localStorage.setItem(STORAGE_KEY, JSON.stringify(state));
    } catch (e) {
        console.error("Error writing localStorage", e);
    }
}

// Cloud Helper for POST transactions
async function cloudPost(route, item) {
    try {
        const res = await fetch(route, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(item)
        });
        if (!res.ok) console.error(`Sync: Cloud POST to '${route}' failed.`);
    } catch (e) {
        console.warn(`Sync: POST '${route}' cached locally (offline).`, e);
    }
    saveState(); // Update local backup cache
}

// Cloud Helper for DELETE transactions
async function cloudDelete(route, id) {
    try {
        const res = await fetch(`${route}?id=${encodeURIComponent(id)}`, {
            method: 'DELETE'
        });
        if (!res.ok) console.error(`Sync: Cloud DELETE from '${route}' failed.`);
    } catch (e) {
        console.warn(`Sync: DELETE '${route}' cached locally (offline).`, e);
    }
    saveState(); // Update local backup cache
}

// Check first use block
function checkFirstUseBlock() {
    const blocker = document.getElementById('first-use-blocker');
    if (state.staff.length === 0) {
        blocker.classList.remove('hidden');
    } else {
        blocker.classList.add('hidden');
    }
}

// Generate demo data for immediate preview
function initializeDemoData() {
    state = {
        staff: [
            { id: "st-1", name: "สมชาย ใจดี", phone: "081-234-5678", role: "Senior Broker", registeredAt: "2026-05-01" },
            { id: "st-2", name: "วิภา รักษ์ห้อง", phone: "082-987-6543", role: "Agent Expert", registeredAt: "2026-05-10" },
            { id: "st-3", name: "กฤษดา นักปิด", phone: "089-111-2222", role: "Junior Broker", registeredAt: "2026-05-15" }
        ],
        cases: [
            { id: "cs-1", name: "คุณอนันต์ พาณิชย์", contact: "085-555-6677", type: "ซื้อ", budget: 7500000, project: "Plum Condo Town", date: getTodayString(), ownerId: "st-1", notes: "หาบ้านเดี่ยวหรือคอนโด 2 ห้องนอน เลี้ยงสัตว์ได้", status: "นัดดูห้อง" },
            { id: "cs-2", name: "คุณพรทิพย์ เจริญ", contact: "083-444-5555", type: "เช่า", budget: 18000, project: "Life Asoke Hype", date: getTodayString(), ownerId: "st-2", notes: "ต้องการสัญญาเช่า 1 ปี ด่วน ย้ายเข้าสิ้นเดือน", status: "ส่งห้องแล้ว" },
            { id: "cs-3", name: "คุณชูเกียรติ มั่งมี", contact: "090-333-2211", type: "ซื้อ", budget: 12000000, project: "Rhythm Sukhumvit 36", date: "2026-05-28", ownerId: "st-1", notes: "ต้องการติดรถไฟฟ้า พร้อมส่วนกลางขนาดใหญ่", status: "จองแล้ว" },
            { id: "cs-4", name: "คุณรวีวรรณ แสนสุข", contact: "081-777-8899", type: "เช่า", budget: 25000, project: "IdeO Q Chula", date: "2026-05-29", ownerId: "st-3", notes: "สัญญา 2 ปี ใกล้ MRT สามย่าน", status: "จองแล้ว" },
            { id: "cs-5", name: "คุณปิยะพร แวววาว", contact: "086-123-4567", type: "เช่า", budget: 12000, project: "Elio Del Nest", date: "2026-05-25", ownerId: "st-2", notes: "งบประหยัด สตูดิโอ", status: "หลุด / ไม่สนใจ" }
        ],
        rentals: [
            { id: "rt-1", caseId: "cs-4", propertyId: "RT-10023", room: "Ideo Q Chula / Room 402 / Floor 12 / Building B", price: 25000, startDate: "2026-06-01", finderClientId: "st-3", finderRoomId: "st-3", showroomAgentId: "st-2", commissionTotal: 25000, evidenceBase64: "", closedAt: "2026-05-30" }
        ],
        sales: [
            { id: "sl-1", caseId: "cs-3", propertyId: "SL-50921", salePrice: 12000000, companyRevenue: 360000, transferDate: "2026-06-15", stakeholders: [{ staffId: "st-1", commission: 120000 }, { staffId: "st-2", commission: 60000 }], evidenceBase64: "", closedAt: "2026-05-29" }
        ]
    };
    saveState();
}

/* ==========================================================================
   UTILITY HELPER FUNCTIONS
   ========================================================================== */
function getTodayString() {
    const today = new Date();
    const yyyy = today.getFullYear();
    const mm = String(today.getMonth() + 1).padStart(2, '0');
    const dd = String(today.getDate()).padStart(2, '0');
    return `${yyyy}-${mm}-${dd}`;
}

function formatCurrency(val) {
    return new Intl.NumberFormat('th-TH', { style: 'currency', currency: 'THB', maximumFractionDigits: 0 }).format(val);
}

function formatThaiDate(dateStr) {
    if (!dateStr) return "-";
    const months = ["ม.ค.", "ก.พ.", "มี.ค.", "เม.ย.", "พ.ค.", "มิ.ย.", "ก.ค.", "ส.ค.", "ก.ย.", "ต.ค.", "พ.ย.", "ธ.ค."];
    const parts = dateStr.split('-');
    if (parts.length !== 3) return dateStr;
    const year = parseInt(parts[0]) + 543; // Thai Buddhist Year
    const month = months[parseInt(parts[1]) - 1];
    const day = parseInt(parts[2]);
    return `${day} ${month} ${year}`;
}

function generateUID(prefix) {
    return `${prefix}-${Math.random().toString(36).substr(2, 9)}`;
}

// Chart.js global config variables
let volChartInstance = null;
let commChartInstance = null;

/* ==========================================================================
   NAVIGATION & UI EVENT ROUTING
   ========================================================================== */
document.addEventListener("DOMContentLoaded", () => {
    loadState();
    lucide.createIcons();
    initDateDisplay();
    checkFirstUseBlock();
    
    // Set default date in forms
    const caseDateInput = document.getElementById('case-date');
    if (caseDateInput) caseDateInput.value = getTodayString();
    
    const rentDateInput = document.getElementById('rent-start-date');
    if (rentDateInput) rentDateInput.value = getTodayString();
    
    const saleDateInput = document.getElementById('sale-transfer-date');
    if (saleDateInput) saleDateInput.value = getTodayString();

    // Trigger primary dropdown loads and renders
    updateAllDropdowns();
    switchTab('dashboard');

    // Sidebar navigation click
    document.querySelectorAll('.nav-item').forEach(item => {
        item.addEventListener('click', (e) => {
            e.preventDefault();
            const tabName = item.getAttribute('data-tab');
            switchTab(tabName);
        });
    });

    // Form Event Listeners
    document.getElementById('first-use-form').addEventListener('submit', handleFirstUseSubmit);
    document.getElementById('case-entry-form').addEventListener('submit', handleCaseSubmit);
    document.getElementById('rental-closing-form').addEventListener('submit', handleRentalSubmit);
    document.getElementById('sale-closing-form').addEventListener('submit', handleSaleSubmit);
    document.getElementById('staff-entry-form').addEventListener('submit', handleStaffSubmit);

    // Form Clear Client Button
    document.getElementById('btn-clear-case').addEventListener('click', clearCaseForm);

    // Close Details Popup Modals
    document.getElementById('btn-close-pop').addEventListener('click', () => {
        document.getElementById('customer-popup-modal').classList.add('hidden');
    });

    // Dynamic stakeholder add in sales
    document.getElementById('btn-add-sale-stakeholder').addEventListener('click', addSaleStakeholderPill);

    // Search case filters
    document.getElementById('search-client').addEventListener('input', () => renderClientsTable());
    document.querySelectorAll('.filter-pill').forEach(pill => {
        pill.addEventListener('click', () => {
            document.querySelectorAll('.filter-pill').forEach(p => p.classList.remove('active'));
            pill.classList.add('active');
            renderClientsTable();
        });
    });

    // Global print trigger
    document.getElementById('btn-global-print').addEventListener('click', handleGlobalPrint);

    // Edit Client from within Popup Modal
    document.getElementById('btn-pop-edit-client').addEventListener('click', triggerClientEdit);
    document.getElementById('btn-pop-print-invoice').addEventListener('click', handleClientInvoicePrint);

    // File Drag/Drop Base64 setups
    setupDropzone('rental-dropzone', 'rent-evidence-file', 'rent-file-name');
    setupDropzone('sale-dropzone', 'sale-evidence-file', 'sale-file-name');

    // Automatically calculate commission total in rentals closing based on value change
    document.getElementById('rent-price').addEventListener('input', (e) => {
        // Normally rental commission = 1 month rent
        document.getElementById('rent-commission-total').value = e.target.value;
        distributeRentCommissionSplits(parseFloat(e.target.value) || 0);
    });

    document.querySelectorAll('.rental-comm-input').forEach(input => {
        input.addEventListener('input', () => {
            // Recalc sum
            let sum = 0;
            document.querySelectorAll('.rental-comm-input').forEach(i => sum += parseFloat(i.value) || 0);
            document.getElementById('rent-commission-total').value = sum;
        });
    });
});

// Setup current date display
function initDateDisplay() {
    const today = new Date();
    const options = { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' };
    const dateText = today.toLocaleDateString('th-TH', options);
    document.getElementById('current-date-text').textContent = dateText;
}

// Dynamic Tab Switching UI router
function switchTab(tabName) {
    // Nav links update
    document.querySelectorAll('.nav-item').forEach(item => {
        if (item.getAttribute('data-tab') === tabName) {
            item.classList.add('active');
        } else {
            item.classList.remove('active');
        }
    });

    // Sub views update
    document.querySelectorAll('.tab-panel').forEach(panel => {
        if (panel.id === `tab-${tabName}`) {
            panel.classList.add('active');
        } else {
            panel.classList.remove('active');
        }
    });

    // Set page subtitle dynamically
    const titleMap = {
        'dashboard': { title: "ภาพรวมทั้งหมด", sub: "แดชบอร์ดความเคลื่อนไหว เคสอสังหาริมทรัพย์ และสถิติค่าคอมมิชชั่นประจำวัน" },
        'cases': { title: "ลงทะเบียนเคส / รายละเอียดลูกค้า", sub: "กรอกข้อมูลเคสใหม่ ทักแชท ตารางบริหารรายชื่อผู้สนใจเช่า/ซื้ออสังหาริมทรัพย์" },
        'tracking': { title: "กระดานติดตามสถานะเคสลูกค้า", sub: "ติดตามสเตจความคืบหน้าของลูกค้าตั้งแต่เคสใหม่ไปจนถึงโอนย้ายสำเร็จ" },
        'rentals': { title: "ลงสัญญาปิดเคสงานเช่า", sub: "บันทึกยอดสัญญา ค่าเช่า และการกระจายแบ่งปันสัดส่วนค่าคอมมิชชั่นของทีมงานปล่อยเช่า" },
        'sales': { title: "ลงบันทึกปิดเคสงานขาย", sub: "บันทึกรายละเอียดราคาโอน รายได้บริษัท สัดส่วนพนักงานรับส่วนแบ่งค่าคอมยอดโอน" },
        'staff': { title: "ทะเบียนรายชื่อพนักงาน", sub: "รายชื่อตัวแทนขายและนายหน้าพนักงานเพื่อนำไปเป็นดรอปดาวน์เลือกในทุกแท็บหน้าต่าง" }
    };

    if (titleMap[tabName]) {
        document.getElementById('page-title').textContent = titleMap[tabName].title;
        document.getElementById('page-subtitle').textContent = titleMap[tabName].sub;
    }

    // Trigger tab specific redraws
    if (tabName === 'dashboard') {
        renderDashboard();
    } else if (tabName === 'cases') {
        renderClientsTable();
    } else if (tabName === 'tracking') {
        renderTrackingBoard();
    } else if (tabName === 'rentals') {
        renderClosedRentals();
    } else if (tabName === 'sales') {
        renderClosedSales();
    } else if (tabName === 'staff') {
        renderStaffTable();
    }
}

// Distribute rental commission split defaults automatically to make it easy
function distributeRentCommissionSplits(totalComm) {
    // If we have total rental commission, we can split it like:
    // Client Finder = 40%, Room Finder = 40%, Showroom Agent = 20%
    const finderClient = Math.round(totalComm * 0.40);
    const finderRoom = Math.round(totalComm * 0.40);
    const showroom = Math.round(totalComm * 0.20);
    
    document.getElementById('rent-commission-finder-client').value = finderClient;
    document.getElementById('rent-commission-finder-room').value = finderRoom;
    document.getElementById('rent-commission-showroom-agent').value = showroom;
}

// Setup custom file drops
function setupDropzone(zoneId, inputId, labelId) {
    const dropzone = document.getElementById(zoneId);
    const input = document.getElementById(inputId);
    const label = document.getElementById(labelId);

    if (!dropzone || !input || !label) return;

    dropzone.addEventListener('click', () => input.click());

    dropzone.addEventListener('dragover', (e) => {
        e.preventDefault();
        dropzone.style.borderColor = 'var(--accent-blue)';
        dropzone.style.background = 'rgba(59, 130, 246, 0.05)';
    });

    dropzone.addEventListener('dragleave', () => {
        dropzone.style.borderColor = 'var(--border-color)';
        dropzone.style.background = 'rgba(255, 255, 255, 0.01)';
    });

    dropzone.addEventListener('drop', (e) => {
        e.preventDefault();
        dropzone.style.borderColor = 'var(--border-color)';
        dropzone.style.background = 'rgba(255, 255, 255, 0.01)';
        
        if (e.dataTransfer.files.length) {
            input.files = e.dataTransfer.files;
            handleFileSelection(input.files[0], label);
        }
    });

    input.addEventListener('change', () => {
        if (input.files.length) {
            handleFileSelection(input.files[0], label);
        }
    });
}

function handleFileSelection(file, labelElement) {
    if (!file) return;
    if (file.size > 1.2 * 1024 * 1024) {
        alert("ไฟล์มีขนาดใหญ่เกิน 1MB กรุณาย่อขนาดรูปภาพก่อนอัปโหลดเพื่อลดขนาดฐานข้อมูล LocalStorage");
        return;
    }
    labelElement.textContent = `${file.name} (${(file.size/1024).toFixed(1)} KB)`;
}

// Convert files to base64 promise wrapper
function getBase64(file) {
    return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.readAsDataURL(file);
        reader.onload = () => resolve(reader.result);
        reader.onerror = error => reject(error);
    });
}

/* ==========================================================================
   DROPDOWN CONTROLLER DYNAMICS
   ========================================================================== */
function updateAllDropdowns() {
    const activeStaff = state.staff;
    const ownersSelect = document.getElementById('case-owner');
    
    const rentFinderClient = document.getElementById('rent-finder-client');
    const rentFinderRoom = document.getElementById('rent-finder-room');
    const rentShowroomAgent = document.getElementById('rent-showroom-agent');
    
    const saleTempStaff = document.getElementById('sale-temp-staff');
    
    // Clear & Rebuild Owner Selects
    let staffOptions = '<option value="" disabled selected>-- เลือกพนักงาน --</option>';
    activeStaff.forEach(st => {
        staffOptions += `<option value="${st.id}">${st.name} (${st.role})</option>`;
    });

    if (ownersSelect) ownersSelect.innerHTML = staffOptions;
    if (rentFinderClient) rentFinderClient.innerHTML = staffOptions;
    if (rentFinderRoom) rentFinderRoom.innerHTML = staffOptions;
    if (rentShowroomAgent) rentShowroomAgent.innerHTML = staffOptions;
    if (saleTempStaff) saleTempStaff.innerHTML = staffOptions;

    // Refresh closed clients selects
    updateClosedClientsDropdowns();
    
    // Sidebar active count
    document.getElementById('sidebar-staff-count').textContent = `พนักงาน: ${state.staff.length} คน`;
}

function updateClosedClientsDropdowns() {
    const rentClientSelect = document.getElementById('rent-client-select');
    const saleClientSelect = document.getElementById('sale-client-select');

    if (!rentClientSelect || !saleClientSelect) return;

    // Rental closing candidate selection: status = "จองแล้ว" + type = "เช่า" + not already closed
    const rentCandidates = state.cases.filter(c => 
        c.status === "จองแล้ว" && 
        c.type === "เช่า" && 
        !state.rentals.some(r => r.caseId === c.id)
    );

    let rentOpt = '<option value="" disabled selected>-- เลือกเคสที่ทำการจองสำเร็จ --</option>';
    rentCandidates.forEach(c => {
        rentOpt += `<option value="${c.id}">${c.name} - โครงการ: ${c.project} (งบ ฿${c.budget.toLocaleString()})</option>`;
    });
    rentClientSelect.innerHTML = rentOpt;

    // Sales closing candidate selection: status = "จองแล้ว" + type = "ซื้อ" + not already closed
    const saleCandidates = state.cases.filter(c => 
        c.status === "จองแล้ว" && 
        c.type === "ซื้อ" && 
        !state.sales.some(s => s.caseId === c.id)
    );

    let saleOpt = '<option value="" disabled selected>-- เลือกเคสที่ทำการจองสำเร็จ --</option>';
    saleCandidates.forEach(c => {
        saleOpt += `<option value="${c.id}">${c.name} - โครงการ: ${c.project} (งบ ฿${c.budget.toLocaleString()})</option>`;
    });
    saleClientSelect.innerHTML = saleOpt;
}

/* ==========================================================================
   SUBMISSIONS & CRUD ACTIONS
   ========================================================================== */

// First use screen submit handler
function handleFirstUseSubmit(e) {
    e.preventDefault();
    const name = document.getElementById('first-staff-name').value.trim();
    const phone = document.getElementById('first-staff-phone').value.trim();
    const role = document.getElementById('first-staff-role').value.trim();

    if (!name || !phone || !role) return;

    const newStaff = {
        id: generateUID('st'),
        name,
        phone,
        role,
        registeredAt: getTodayString()
    };

    state.staff.push(newStaff);
    saveState();
    
    // Clear & hide
    document.getElementById('first-use-form').reset();
    checkFirstUseBlock();
    updateAllDropdowns();
    switchTab('dashboard');
}

// Case submitting (Add or Update)
function handleCaseSubmit(e) {
    e.preventDefault();
    const editId = document.getElementById('case-edit-id').value;
    const name = document.getElementById('case-client-name').value.trim();
    const contact = document.getElementById('case-client-contact').value.trim();
    const type = document.getElementById('case-type').value;
    const budget = parseFloat(document.getElementById('case-budget').value) || 0;
    const project = document.getElementById('case-project').value.trim();
    const date = document.getElementById('case-date').value;
    const ownerId = document.getElementById('case-owner').value;
    const status = document.getElementById('case-status').value;
    const notes = document.getElementById('case-notes').value.trim();

    if (editId) {
        // Edit Mode
        const caseIdx = state.cases.findIndex(c => c.id === editId);
        if (caseIdx !== -1) {
            state.cases[caseIdx] = {
                ...state.cases[caseIdx],
                name, contact, type, budget, project, date, ownerId, status, notes
            };
        }
    } else {
        // Add Mode
        const newCase = {
            id: generateUID('cs'),
            name, contact, type, budget, project, date, ownerId, status, notes
        };
        state.cases.push(newCase);
    }

    saveState();
    clearCaseForm();
    updateClosedClientsDropdowns();
    renderClientsTable();
    renderTrackingBoard();
}

function clearCaseForm() {
    document.getElementById('case-entry-form').reset();
    document.getElementById('case-edit-id').value = '';
    document.getElementById('btn-save-case').textContent = 'บันทึกข้อมูลเคส';
    document.getElementById('case-date').value = getTodayString();
    
    // Retain registered owner dropdown selection default
    if (state.staff.length > 0) {
        document.getElementById('case-owner').selectedIndex = 1;
    }
}

// Staff submitting
function handleStaffSubmit(e) {
    e.preventDefault();
    const name = document.getElementById('staff-name').value.trim();
    const phone = document.getElementById('staff-phone').value.trim();
    const role = document.getElementById('staff-role').value.trim();

    if (!name || !phone || !role) return;

    const newStaff = {
        id: generateUID('st'),
        name,
        phone,
        role,
        registeredAt: getTodayString()
    };

    state.staff.push(newStaff);
    saveState();
    
    document.getElementById('staff-entry-form').reset();
    updateAllDropdowns();
    renderStaffTable();
}

// Rental close submit
async function handleRentalSubmit(e) {
    e.preventDefault();
    const caseId = document.getElementById('rent-client-select').value;
    const propertyId = document.getElementById('rent-property-id').value.trim();
    const room = document.getElementById('rent-room-details').value.trim();
    const price = parseFloat(document.getElementById('rent-price').value) || 0;
    const startDate = document.getElementById('rent-start-date').value;

    const finderClientId = document.getElementById('rent-finder-client').value;
    const fClientComm = parseFloat(document.getElementById('rent-commission-finder-client').value) || 0;

    const finderRoomId = document.getElementById('rent-finder-room').value;
    const fRoomComm = parseFloat(document.getElementById('rent-commission-finder-room').value) || 0;

    const showroomAgentId = document.getElementById('rent-showroom-agent').value;
    const showroomComm = parseFloat(document.getElementById('rent-commission-showroom-agent').value) || 0;

    const commissionTotal = parseFloat(document.getElementById('rent-commission-total').value) || 0;

    const fileInput = document.getElementById('rent-evidence-file');
    let evidenceBase64 = "";
    if (fileInput.files.length > 0) {
        evidenceBase64 = await getBase64(fileInput.files[0]);
    }

    const newRental = {
        id: generateUID('rt'),
        caseId,
        propertyId,
        room,
        price,
        startDate,
        finderClientId,
        finderRoomId,
        showroomAgentId,
        commissionTotal,
        // Individual splits stored inside for precise dashboard accounting
        splits: [
            { staffId: finderClientId, commission: fClientComm },
            { staffId: finderRoomId, commission: fRoomComm },
            { staffId: showroomAgentId, commission: showroomComm }
        ],
        evidenceBase64,
        closedAt: getTodayString()
    };

    state.rentals.push(newRental);

    // Dynamically update linked case status to reserved/closed
    const targetCase = state.cases.find(c => c.id === caseId);
    if (targetCase) {
        targetCase.status = "จองแล้ว"; 
    }

    saveState();
    document.getElementById('rental-closing-form').reset();
    document.getElementById('rent-file-name').textContent = "ไม่มีไฟล์แนบ";
    updateClosedClientsDropdowns();
    renderClosedRentals();
}

// Sales close submit
async function handleSaleSubmit(e) {
    e.preventDefault();
    const caseId = document.getElementById('sale-client-select').value;
    const propertyId = document.getElementById('sale-property-id').value.trim();
    const salePrice = parseFloat(document.getElementById('sale-price').value) || 0;
    const companyRevenue = parseFloat(document.getElementById('sale-company-revenue').value) || 0;
    const transferDate = document.getElementById('sale-transfer-date').value;

    if (tempSaleStakeholders.length === 0) {
        alert("กรุณาระบุรายชื่อผู้ได้รับส่วนแบ่งอย่างน้อย 1 คนก่อนบันทึกปิดเคสงานขาย");
        return;
    }

    const fileInput = document.getElementById('sale-evidence-file');
    let evidenceBase64 = "";
    if (fileInput.files.length > 0) {
        evidenceBase64 = await getBase64(fileInput.files[0]);
    }

    const newSale = {
        id: generateUID('sl'),
        caseId,
        propertyId,
        salePrice,
        companyRevenue,
        transferDate,
        stakeholders: [...tempSaleStakeholders],
        evidenceBase64,
        closedAt: getTodayString()
    };

    state.sales.push(newSale);

    // Update case status
    const targetCase = state.cases.find(c => c.id === caseId);
    if (targetCase) {
        targetCase.status = "จองแล้ว";
    }

    saveState();
    
    // Reset temp stack & UI
    tempSaleStakeholders = [];
    document.getElementById('sale-closing-form').reset();
    document.getElementById('sale-file-name').textContent = "ไม่มีไฟล์แนบ";
    document.getElementById('sale-stakeholders-container').innerHTML = '<p class="placeholder-text-box">ยังไม่ได้ระบุผู้ได้รับส่วนแบ่งค่าคอมฯ</p>';
    
    updateClosedClientsDropdowns();
    renderClosedSales();
}

// Manage dynamic stakeholder row arrays in closing sales form
function addSaleStakeholderPill() {
    const staffId = document.getElementById('sale-temp-staff').value;
    const commAmt = parseFloat(document.getElementById('sale-temp-comm').value) || 0;

    if (!staffId) {
        alert("กรุณาเลือกพนักงาน");
        return;
    }
    if (commAmt <= 0) {
        alert("กรุณากรอกจำนวนเงินค่าคอมมิชชั่นที่ถูกต้อง");
        return;
    }

    const staffObj = state.staff.find(st => st.id === staffId);
    if (!staffObj) return;

    // Check duplicates
    if (tempSaleStakeholders.some(st => st.staffId === staffId)) {
        alert("พนักงานคนนี้มีรายชื่อได้รับส่วนแบ่งอยู่แล้วในรายการนี้");
        return;
    }

    tempSaleStakeholders.push({
        staffId: staffId,
        commission: commAmt,
        name: staffObj.name
    });

    // Clear temp input
    document.getElementById('sale-temp-comm').value = '';

    renderSaleStakeholdersList();
}

function renderSaleStakeholdersList() {
    const container = document.getElementById('sale-stakeholders-container');
    if (tempSaleStakeholders.length === 0) {
        container.innerHTML = '<p class="placeholder-text-box">ยังไม่ได้ระบุผู้ได้รับส่วนแบ่งค่าคอมฯ</p>';
        return;
    }

    container.innerHTML = '';
    tempSaleStakeholders.forEach((sh, index) => {
        const pill = document.createElement('div');
        pill.className = 'stakeholder-pill';
        pill.innerHTML = `
            <span class="st-name">${sh.name}</span>
            <div class="flex-center" style="flex-direction:row; gap: 0.5rem;">
                <span class="st-comm">${formatCurrency(sh.commission)}</span>
                <button type="button" class="btn-remove-pill" data-index="${index}">
                    <i data-lucide="trash-2"></i>
                </button>
            </div>
        `;
        container.appendChild(pill);
    });

    lucide.createIcons();

    // Wire deletes
    container.querySelectorAll('.btn-remove-pill').forEach(btn => {
        btn.addEventListener('click', (e) => {
            const idx = parseInt(btn.getAttribute('data-index'));
            tempSaleStakeholders.splice(idx, 1);
            renderSaleStakeholdersList();
        });
    });
}

/* ==========================================================================
   RENDERING ROUTINES & DOM INJECTIONS
   ========================================================================== */

// 1. Dashboard Tab Renderers
function renderDashboard() {
    const todayStr = getTodayString();
    
    // KPI 1: New Cases today
    const newCasesToday = state.cases.filter(c => c.date === todayStr).length;
    document.getElementById('kpi-new-today').textContent = newCasesToday;

    // KPI 2: Cases In Progress
    const inProgressStages = ['คุยแล้ว', 'ส่งห้องแล้ว', 'นัดดูห้อง', 'รอตัดสินใจ'];
    const activeTracking = state.cases.filter(c => inProgressStages.includes(c.status)).length;
    document.getElementById('kpi-in-progress').textContent = activeTracking;

    // KPI 3: Room Viewings
    const viewings = state.cases.filter(c => c.status === "นัดดูห้อง").length;
    document.getElementById('kpi-viewings').textContent = viewings;

    // KPI 4: Closed rentals count
    document.getElementById('kpi-rent-closed').textContent = state.rentals.length;

    // KPI 5: Closed sales count
    document.getElementById('kpi-sale-closed').textContent = state.sales.length;

    // Financial calculations
    let totalSaleVolume = 0;
    let totalCompanyRevenue = 0;

    state.sales.forEach(sl => {
        totalSaleVolume += sl.salePrice;
        totalCompanyRevenue += sl.companyRevenue;
    });

    state.rentals.forEach(rt => {
        // For rentals: total volume can be sum of rents, revenue is total rental commissions
        totalCompanyRevenue += rt.commissionTotal;
    });

    document.getElementById('dash-total-sales-volume').textContent = formatCurrency(totalSaleVolume);
    document.getElementById('dash-total-revenue').textContent = formatCurrency(totalCompanyRevenue);

    // Compute dynamic commission maps per employee
    const staffStats = state.staff.map(st => {
        let rentCount = 0;
        let saleCount = 0;
        let rentComm = 0;
        let saleComm = 0;

        // Trace rentals where staff was involved
        state.rentals.forEach(rt => {
            const involved = rt.splits || [
                { staffId: rt.finderClientId, commission: rt.commissionTotal * 0.4 },
                { staffId: rt.finderRoomId, commission: rt.commissionTotal * 0.4 },
                { staffId: rt.showroomAgentId, commission: rt.commissionTotal * 0.2 }
            ];

            involved.forEach(sh => {
                if (sh.staffId === st.id) {
                    rentComm += sh.commission;
                    rentCount += 0.33; // Fractional contribution or counts as case closed
                }
            });
        });

        // Trace sales where staff was involved
        state.sales.forEach(sl => {
            sl.stakeholders.forEach(sh => {
                if (sh.staffId === st.id) {
                    saleComm += sh.commission;
                    saleCount += 0.5; // Fractional closed count
                }
            });
        });

        // Round case counts for clean UI display
        return {
            id: st.id,
            name: st.name,
            role: st.role,
            rentCount: Math.ceil(rentCount),
            saleCount: Math.ceil(saleCount),
            rentComm,
            saleComm,
            totalComm: rentComm + saleComm
        };
    });

    // Populate dashboard table
    const tableBody = document.getElementById('dashboard-staff-table-body');
    tableBody.innerHTML = '';

    if (staffStats.length === 0) {
        tableBody.innerHTML = `<tr><td colspan="7" class="text-center text-muted">ยังไม่พบข้อมูลพนักงานในระบบ</td></tr>`;
    } else {
        staffStats.forEach(st => {
            const tr = document.createElement('tr');
            tr.innerHTML = `
                <td><strong>${st.name}</strong></td>
                <td><span class="text-muted">${st.role}</span></td>
                <td>${st.rentCount} เคส</td>
                <td>${st.saleCount} เคส</td>
                <td class="text-emerald">${formatCurrency(st.rentComm)}</td>
                <td class="text-emerald">${formatCurrency(st.saleComm)}</td>
                <td><strong class="text-emerald">${formatCurrency(st.totalComm)}</strong></td>
            `;
            tableBody.appendChild(tr);
        });
    }

    // Render visual charts (Chart.js)
    renderCharts(staffStats);
}

function renderCharts(staffStats) {
    // 1. Monthly Volume Trend Chart (Mock trends by combining sales + rentals)
    const volCtx = document.getElementById('monthlyVolumeChart').getContext('2d');
    if (volChartInstance) {
        volChartInstance.destroy();
    }

    // Grouping revenues dynamically by months
    const monthlyGroups = {};
    const monthsName = ["ม.ค.", "ก.พ.", "มี.ค.", "เม.ย.", "พ.ค.", "มิ.ย.", "ก.ค.", "ส.ค.", "ก.ย.", "ต.ค.", "พ.ย.", "ธ.ค."];
    
    // Init last 6 months
    const today = new Date();
    for (let i = 5; i >= 0; i--) {
        const d = new Date(today.getFullYear(), today.getMonth() - i, 1);
        const label = `${monthsName[d.getMonth()]} ${String(d.getFullYear() + 543).slice(-2)}`;
        monthlyGroups[label] = { rentalComm: 0, saleRev: 0 };
    }

    // Populate actuals
    state.rentals.forEach(rt => {
        if (!rt.closedAt) return;
        const d = new Date(rt.closedAt);
        const label = `${monthsName[d.getMonth()]} ${String(d.getFullYear() + 543).slice(-2)}`;
        if (monthlyGroups[label] !== undefined) {
            monthlyGroups[label].rentalComm += rt.commissionTotal;
        }
    });

    state.sales.forEach(sl => {
        if (!sl.transferDate) return;
        const d = new Date(sl.transferDate);
        const label = `${monthsName[d.getMonth()]} ${String(d.getFullYear() + 543).slice(-2)}`;
        if (monthlyGroups[label] !== undefined) {
            monthlyGroups[label].saleRev += sl.companyRevenue;
        }
    });

    const labels = Object.keys(monthlyGroups);
    const rentalData = labels.map(l => monthlyGroups[l].rentalComm);
    const saleData = labels.map(l => monthlyGroups[l].saleRev);

    volChartInstance = new Chart(volCtx, {
        type: 'bar',
        data: {
            labels: labels,
            datasets: [
                {
                    label: 'ค่าคอมมิชชั่นเช่ารวม',
                    data: rentalData,
                    backgroundColor: 'rgba(6, 182, 212, 0.75)',
                    borderColor: 'rgb(6, 182, 212)',
                    borderWidth: 1,
                    borderRadius: 4
                },
                {
                    label: 'รายได้คอมมิชชั่นขายรวม',
                    data: saleData,
                    backgroundColor: 'rgba(242, 168, 24, 0.75)',
                    borderColor: 'rgb(242, 168, 24)',
                    borderWidth: 1,
                    borderRadius: 4
                }
            ]
        },
        options: {
            responsive: true,
            maintainAspectRatio: false,
            plugins: {
                legend: { labels: { color: '#9ca3af' } }
            },
            scales: {
                x: { ticks: { color: '#9ca3af' }, grid: { color: 'rgba(255, 255, 255, 0.05)' } },
                y: { ticks: { color: '#9ca3af' }, grid: { color: 'rgba(255, 255, 255, 0.05)' } }
            }
        }
    });

    // 2. Commission split pie chart
    const commCtx = document.getElementById('staffCommissionChart');
    const emptyComm = document.getElementById('commissions-empty');

    const activeComms = staffStats.filter(st => st.totalComm > 0);

    if (activeComms.length === 0) {
        commCtx.classList.add('hidden');
        emptyComm.classList.remove('hidden');
    } else {
        commCtx.classList.remove('hidden');
        emptyComm.classList.add('hidden');

        if (commChartInstance) {
            commChartInstance.destroy();
        }

        commChartInstance = new Chart(commCtx.getContext('2d'), {
            type: 'doughnut',
            data: {
                labels: activeComms.map(st => st.name),
                datasets: [{
                    data: activeComms.map(st => st.totalComm),
                    backgroundColor: [
                        '#f2a818', '#0ea5e9', '#10b981', '#f43f5e', '#8b5cf6', '#6b7280'
                    ],
                    borderWidth: 1,
                    borderColor: 'rgba(17, 24, 39, 0.8)'
                }]
            },
            options: {
                responsive: true,
                maintainAspectRatio: false,
                plugins: {
                    legend: {
                        position: 'right',
                        labels: { color: '#9ca3af' }
                    }
                }
            }
        });
    }
}

// 2. Client Database / Register Tab Renderers
function renderClientsTable() {
    const searchVal = document.getElementById('search-client').value.toLowerCase().trim();
    const filterType = document.querySelector('.filter-pill.active').getAttribute('data-filter');
    const tbody = document.getElementById('clients-table-body');
    tbody.innerHTML = '';

    let filtered = state.cases;

    // Filter type search
    if (filterType !== 'all') {
        filtered = filtered.filter(c => c.type === filterType);
    }

    // Query text search
    if (searchVal) {
        filtered = filtered.filter(c => 
            c.name.toLowerCase().includes(searchVal) ||
            c.contact.toLowerCase().includes(searchVal) ||
            c.project.toLowerCase().includes(searchVal) ||
            c.status.toLowerCase().includes(searchVal)
        );
    }

    // Sort descending by date
    filtered.sort((a,b) => b.date.localeCompare(a.date));

    if (filtered.length === 0) {
        tbody.innerHTML = `<tr><td colspan="8" class="text-center text-muted">ไม่พบคลิกเคสลูกค้าตรงตามเงื่อนไขค้นหา</td></tr>`;
        return;
    }

    filtered.forEach(c => {
        const staff = state.staff.find(s => s.id === c.ownerId);
        const ownerName = staff ? staff.name : 'ไม่ระบุ';
        const typeClass = c.type === 'เช่า' ? 'rent' : 'sale';
        const statusBadge = getStatusBadgeClass(c.status);

        const tr = document.createElement('tr');
        tr.innerHTML = `
            <td>${formatThaiDate(c.date)}</td>
            <td><strong>${c.name}</strong></td>
            <td><span class="type-tag ${typeClass}">${c.type}</span></td>
            <td>${c.project}</td>
            <td><strong>${formatCurrency(c.budget)}</strong></td>
            <td>${ownerName}</td>
            <td><span class="status-badge ${statusBadge}">${c.status}</span></td>
            <td class="actions-col">
                <div class="action-btns" onclick="event.stopPropagation()">
                    <button class="btn-icon view" title="ดูประวัติลูกค้า" data-id="${c.id}"><i data-lucide="eye"></i></button>
                    <button class="btn-icon edit" title="แก้ไขเคส" data-id="${c.id}"><i data-lucide="edit-3"></i></button>
                    <button class="btn-icon delete" title="ลบเคส" data-id="${c.id}"><i data-lucide="trash-2"></i></button>
                </div>
            </td>
        `;

        // Row Click opens full detailed profile popup
        tr.addEventListener('click', () => {
            showCustomerDetailsPopup(c.id);
        });

        tbody.appendChild(tr);
    });

    lucide.createIcons();

    // Wire actions
    tbody.querySelectorAll('.btn-icon.view').forEach(btn => {
        btn.addEventListener('click', () => showCustomerDetailsPopup(btn.getAttribute('data-id')));
    });

    tbody.querySelectorAll('.btn-icon.edit').forEach(btn => {
        btn.addEventListener('click', () => loadCaseIntoForm(btn.getAttribute('data-id')));
    });

    tbody.querySelectorAll('.btn-icon.delete').forEach(btn => {
        btn.addEventListener('click', () => handleDeleteCase(btn.getAttribute('data-id')));
    });
}

function getStatusBadgeClass(status) {
    const map = {
        'เคสใหม่': 'badge-new',
        'คุยแล้ว': 'badge-talked',
        'ส่งห้องแล้ว': 'badge-sent',
        'นัดดูห้อง': 'badge-viewing',
        'รอตัดสินใจ': 'badge-waiting',
        'จองแล้ว': 'badge-reserved',
        'หลุด / ไม่สนใจ': 'badge-lost'
    };
    return map[status] || 'badge-new';
}

function loadCaseIntoForm(id) {
    const c = state.cases.find(item => item.id === id);
    if (!c) return;

    document.getElementById('case-edit-id').value = c.id;
    document.getElementById('case-client-name').value = c.name;
    document.getElementById('case-client-contact').value = c.contact;
    document.getElementById('case-type').value = c.type;
    document.getElementById('case-budget').value = c.budget;
    document.getElementById('case-project').value = c.project;
    document.getElementById('case-date').value = c.date;
    document.getElementById('case-owner').value = c.ownerId;
    document.getElementById('case-status').value = c.status;
    document.getElementById('case-notes').value = c.notes;

    document.getElementById('btn-save-case').textContent = 'อัปเดตข้อมูลเคส';
    
    // Focus back on form
    document.getElementById('case-client-name').focus();
}

function handleDeleteCase(id) {
    if (confirm("คุณแน่ใจหรือไม่ที่จะต้องการลบเคสลูกค้านี้ออกจากฐานข้อมูลระบบ?")) {
        state.cases = state.cases.filter(c => c.id !== id);
        // Also cleanup closed rentals/sales referencing this case
        state.rentals = state.rentals.filter(r => r.caseId !== id);
        state.sales = state.sales.filter(s => s.caseId !== id);
        
        saveState();
        updateClosedClientsDropdowns();
        renderClientsTable();
        renderTrackingBoard();
    }
}

// 3. Kanban Tracking Board Renderers
function renderTrackingBoard() {
    const columns = ['เคสใหม่', 'คุยแล้ว', 'ส่งห้องแล้ว', 'นัดดูห้อง', 'รอตัดสินใจ', 'จองแล้ว', 'หลุด / ไม่สนใจ'];
    const countBadgeMap = {};

    // Clear board columns
    columns.forEach(col => {
        const el = document.getElementById(`stage-${getStageIdName(col)}-cards`);
        if (el) el.innerHTML = '';
        countBadgeMap[col] = 0;
    });

    let trackingCount = 0;

    state.cases.forEach(c => {
        const colIdName = getStageIdName(c.status);
        const container = document.getElementById(`stage-${colIdName}-cards`);
        if (!container) return;

        countBadgeMap[c.status]++;

        // Count cases that are still in progress for sidebar badge count
        if (['เคสใหม่', 'คุยแล้ว', 'ส่งห้องแล้ว', 'นัดดูห้อง', 'รอตัดสินใจ'].includes(c.status)) {
            trackingCount++;
        }

        const card = document.createElement('div');
        card.className = `kanban-card ${c.type === 'เช่า' ? 'card-rent' : 'card-sale'}`;
        card.innerHTML = `
            <div class="card-header-row">
                <span class="card-client-name">${c.name}</span>
                <span class="type-tag ${c.type === 'เช่า' ? 'rent' : 'sale'}">${c.type}</span>
            </div>
            <p class="card-project"><i data-lucide="map-pin"></i> ${c.project}</p>
            <div class="card-details">
                <span class="card-budget">${formatCurrency(c.budget)}</span>
                <span>${formatThaiDate(c.date)}</span>
            </div>
            
            <div class="quick-move-controls" onclick="event.stopPropagation()">
                <button class="btn-quick-move btn-move-prev" title="ย้ายไปสถานะก่อนหน้า"><i data-lucide="chevron-left"></i></button>
                <button class="btn-quick-move btn-move-next" title="ย้ายไปสถานะถัดไป"><i data-lucide="chevron-right"></i></button>
            </div>
        `;

        card.addEventListener('click', () => {
            showCustomerDetailsPopup(c.id);
        });

        // Wire quick moves
        const prevBtn = card.querySelector('.btn-move-prev');
        const nextBtn = card.querySelector('.btn-move-next');

        const currentStageIdx = columns.indexOf(c.status);

        prevBtn.addEventListener('click', () => {
            if (currentStageIdx > 0) {
                updateCaseStatus(c.id, columns[currentStageIdx - 1]);
            }
        });

        nextBtn.addEventListener('click', () => {
            if (currentStageIdx < columns.length - 1) {
                updateCaseStatus(c.id, columns[currentStageIdx + 1]);
            }
        });

        container.appendChild(card);
    });

    lucide.createIcons();

    // Update Kanban column header count badges
    columns.forEach(col => {
        const columnHeader = document.querySelector(`.kanban-column[data-stage="${col}"] .count-badge`);
        if (columnHeader) {
            columnHeader.textContent = countBadgeMap[col];
        }
    });

    document.getElementById('tracking-count').textContent = trackingCount;
}

function getStageIdName(stage) {
    const map = {
        'เคสใหม่': 'new',
        'คุยแล้ว': 'talked',
        'ส่งห้องแล้ว': 'sent',
        'นัดดูห้อง': 'viewing',
        'รอตัดสินใจ': 'waiting',
        'จองแล้ว': 'reserved',
        'หลุด / ไม่สนใจ': 'lost'
    };
    return map[stage] || 'new';
}

function updateCaseStatus(caseId, newStatus) {
    const c = state.cases.find(item => item.id === caseId);
    if (!c) return;

    c.status = newStatus;
    saveState();
    updateClosedClientsDropdowns();
    renderTrackingBoard();
}

// 4. Closed Rentals Tab Renderers
function renderClosedRentals() {
    const tbody = document.getElementById('rentals-table-body');
    tbody.innerHTML = '';

    if (state.rentals.length === 0) {
        tbody.innerHTML = `<tr><td colspan="8" class="text-center text-muted">ยังไม่มีบันทึกข้อมูลประวัติปิดงานเช่าสำเร็จ</td></tr>`;
        return;
    }

    state.rentals.forEach(rt => {
        const c = state.cases.find(item => item.id === rt.caseId);
        const clientName = c ? c.name : 'ไม่ระบุ';
        
        let fileBadge = '<span class="text-muted">ไม่ได้แนบ</span>';
        if (rt.evidenceBase64) {
            fileBadge = `<span class="evidence-mini-badge"><i data-lucide="image"></i> แนบแล้ว</span>`;
        }

        const tr = document.createElement('tr');
        tr.innerHTML = `
            <td><strong>${rt.propertyId}</strong></td>
            <td>${clientName}</td>
            <td>${rt.room}</td>
            <td><strong>${formatCurrency(rt.price)}</strong></td>
            <td>${formatThaiDate(rt.startDate)}</td>
            <td class="text-emerald"><strong>${formatCurrency(rt.commissionTotal)}</strong></td>
            <td>${fileBadge}</td>
            <td class="actions-col">
                <div class="action-btns">
                    <button class="btn-icon view" title="ดูรายละเอียด" data-id="${rt.caseId}"><i data-lucide="eye"></i></button>
                    <button class="btn-icon delete" title="ลบประวัติ" data-id="${rt.id}"><i data-lucide="trash-2"></i></button>
                </div>
            </td>
        `;

        tbody.appendChild(tr);
    });

    lucide.createIcons();

    // Wire actions
    tbody.querySelectorAll('.btn-icon.view').forEach(btn => {
        btn.addEventListener('click', () => showCustomerDetailsPopup(btn.getAttribute('data-id')));
    });

    tbody.querySelectorAll('.btn-icon.delete').forEach(btn => {
        btn.addEventListener('click', () => handleDeleteClosedRental(btn.getAttribute('data-id')));
    });
}

function handleDeleteClosedRental(id) {
    if (confirm("ต้องการลบประวัติการปิดงานเช่ารายการนี้หรือไม่? (ข้อมูลยอดเงินและประวัติที่เกี่ยวข้องจะหายไปจากแดชบอร์ด)")) {
        const rt = state.rentals.find(r => r.id === id);
        if (rt) {
            // Revert linked case status to RESERVED so it can be re-closed or managed
            const c = state.cases.find(item => item.id === rt.caseId);
            if (c) c.status = "จองแล้ว";
        }
        
        state.rentals = state.rentals.filter(r => r.id !== id);
        saveState();
        updateClosedClientsDropdowns();
        renderClosedRentals();
    }
}

// 5. Closed Sales Tab Renderers
function renderClosedSales() {
    const tbody = document.getElementById('sales-table-body');
    tbody.innerHTML = '';

    if (state.sales.length === 0) {
        tbody.innerHTML = `<tr><td colspan="8" class="text-center text-muted">ยังไม่มีบันทึกข้อมูลประวัติปิดงานขายสำเร็จ</td></tr>`;
        return;
    }

    state.sales.forEach(sl => {
        const c = state.cases.find(item => item.id === sl.caseId);
        const clientName = c ? c.name : 'ไม่ระบุ';

        let stakeholdersStr = "";
        sl.stakeholders.forEach((sh, index) => {
            const staffObj = state.staff.find(s => s.id === sh.staffId);
            const name = staffObj ? staffObj.name : "พนักงาน";
            stakeholdersStr += `${name} (${formatCurrency(sh.commission)})${index < sl.stakeholders.length - 1 ? ', ' : ''}`;
        });

        let fileBadge = '<span class="text-muted">ไม่ได้แนบ</span>';
        if (sl.evidenceBase64) {
            fileBadge = `<span class="evidence-mini-badge"><i data-lucide="image"></i> แนบแล้ว</span>`;
        }

        const tr = document.createElement('tr');
        tr.innerHTML = `
            <td><strong>${sl.propertyId}</strong></td>
            <td>${clientName}</td>
            <td><strong>${formatCurrency(sl.salePrice)}</strong></td>
            <td class="text-emerald"><strong>${formatCurrency(sl.companyRevenue)}</strong></td>
            <td>${formatThaiDate(sl.transferDate)}</td>
            <td><small class="text-muted" style="display:block; max-width:200px; overflow:hidden; text-overflow:ellipsis; white-space:nowrap;" title="${stakeholdersStr}">${stakeholdersStr}</small></td>
            <td>${fileBadge}</td>
            <td class="actions-col">
                <div class="action-btns">
                    <button class="btn-icon view" title="ดูรายละเอียด" data-id="${sl.caseId}"><i data-lucide="eye"></i></button>
                    <button class="btn-icon delete" title="ลบประวัติ" data-id="${sl.id}"><i data-lucide="trash-2"></i></button>
                </div>
            </td>
        `;

        tbody.appendChild(tr);
    });

    lucide.createIcons();

    // Wire actions
    tbody.querySelectorAll('.btn-icon.view').forEach(btn => {
        btn.addEventListener('click', () => showCustomerDetailsPopup(btn.getAttribute('data-id')));
    });

    tbody.querySelectorAll('.btn-icon.delete').forEach(btn => {
        btn.addEventListener('click', () => handleDeleteClosedSale(btn.getAttribute('data-id')));
    });
}

function handleDeleteClosedSale(id) {
    if (confirm("ต้องการลบประวัติการปิดงานขายรายการนี้หรือไม่? (ข้อมูลยอดเงินจะถูกหักออกจากแดชบอร์ดอย่างถาวร)")) {
        const sl = state.sales.find(s => s.id === id);
        if (sl) {
            const c = state.cases.find(item => item.id === sl.caseId);
            if (c) c.status = "จองแล้ว";
        }

        state.sales = state.sales.filter(s => s.id !== id);
        saveState();
        updateClosedClientsDropdowns();
        renderClosedSales();
    }
}

// 6. Staff Registry Tab Renderers
function renderStaffTable() {
    const tbody = document.getElementById('staff-table-body');
    tbody.innerHTML = '';

    if (state.staff.length === 0) {
        tbody.innerHTML = `<tr><td colspan="6" class="text-center text-muted">ยังไม่พบรายชื่อพนักงานที่ลงทะเบียนในระบบ</td></tr>`;
        return;
    }

    state.staff.forEach((st, index) => {
        const tr = document.createElement('tr');
        tr.innerHTML = `
            <td>${index + 1}</td>
            <td><strong>${st.name}</strong></td>
            <td>${st.phone}</td>
            <td><span class="type-tag sale" style="font-weight:500;">${st.role}</span></td>
            <td>${formatThaiDate(st.registeredAt)}</td>
            <td class="actions-col">
                <div class="action-btns">
                    <button class="btn-icon delete" title="ลบพนักงาน" data-id="${st.id}"><i data-lucide="user-minus"></i></button>
                </div>
            </td>
        `;

        tbody.appendChild(tr);
    });

    lucide.createIcons();

    // Wire deletes
    tbody.querySelectorAll('.btn-icon.delete').forEach(btn => {
        btn.addEventListener('click', () => handleDeleteStaff(btn.getAttribute('data-id')));
    });
}

function handleDeleteStaff(id) {
    if (state.staff.length === 1) {
        alert("ขออภัย! ระบบจำเป็นต้องมีพนักงานลงทะเบียนอยู่อย่างน้อย 1 คน เพื่อใช้ดึงข้อมูลฟอร์มดรอปดาวน์พนักงาน");
        return;
    }

    if (confirm("คำเตือน: การลบพนักงานอาจส่งผลกระทบต่อประวัติที่รับค่าคอมมิชชั่นและการมอบหมายเคสของคนๆ นี้ คุณยืนยันจะต้องการลบหรือไม่?")) {
        state.staff = state.staff.filter(st => st.id !== id);
        saveState();
        updateAllDropdowns();
        renderStaffTable();
    }
}

/* ==========================================================================
   POPUP DETAILS MODAL VIEW LOGICS
   ========================================================================== */
let activePopupCaseId = null;

function showCustomerDetailsPopup(caseId) {
    const c = state.cases.find(item => item.id === caseId);
    if (!c) return;

    activePopupCaseId = caseId;

    const staffObj = state.staff.find(st => st.id === c.ownerId);
    const ownerName = staffObj ? staffObj.name : 'ไม่ระบุ';

    document.getElementById('pop-client-name').textContent = c.name;
    document.getElementById('pop-client-type').textContent = c.type;
    document.getElementById('pop-client-type').className = `client-type-tag ${c.type === 'เช่า' ? 'rent' : 'sale'}`;
    
    document.getElementById('pop-client-contact').textContent = c.contact;
    document.getElementById('pop-client-budget').textContent = formatCurrency(c.budget);
    document.getElementById('pop-client-project').textContent = c.project;
    document.getElementById('pop-client-date').textContent = formatThaiDate(c.date);
    document.getElementById('pop-client-owner').textContent = ownerName;
    
    document.getElementById('pop-client-status').textContent = c.status;
    document.getElementById('pop-client-status').className = `detail-value status-badge ${getStatusBadgeClass(c.status)}`;
    
    document.getElementById('pop-client-notes').textContent = c.notes || 'ไม่มีบันทึกข้อความหมายเหตุสำหรับลูกค้ารายนี้';

    // Closing Check details
    const closingRent = state.rentals.find(r => r.caseId === c.id);
    const closingSale = state.sales.find(s => s.caseId === c.id);

    const closingSection = document.getElementById('pop-closing-section');
    const evidenceBox = document.getElementById('pop-evidence-box');
    const evidenceImg = document.getElementById('pop-evidence-img');
    const noEvidence = document.getElementById('pop-no-evidence');

    if (closingRent) {
        closingSection.classList.remove('hidden');
        document.getElementById('pop-close-property-id').textContent = closingRent.propertyId;
        document.getElementById('pop-close-value').textContent = formatCurrency(closingRent.price) + " / เดือน";
        document.getElementById('pop-close-room').textContent = closingRent.room;
        
        document.getElementById('pop-close-agent-lbl-1').textContent = "ผู้พาดูห้อง / หาห้อง";
        
        const showroomAgent = state.staff.find(st => st.id === closingRent.showroomAgentId);
        const roomFinder = state.staff.find(st => st.id === closingRent.finderRoomId);
        document.getElementById('pop-close-agent-val-1').textContent = `${showroomAgent ? showroomAgent.name : '-'} / ${roomFinder ? roomFinder.name : '-'}`;
        
        document.getElementById('pop-close-agent-lbl-2').textContent = "ค่าคอมมิชชั่นรวมบริษัท";
        document.getElementById('pop-close-agent-val-2').textContent = formatCurrency(closingRent.commissionTotal);

        // Attachment handler
        if (closingRent.evidenceBase64) {
            evidenceImg.src = closingRent.evidenceBase64;
            evidenceImg.classList.remove('hidden');
            noEvidence.classList.add('hidden');
        } else {
            evidenceImg.classList.add('hidden');
            noEvidence.classList.remove('hidden');
        }
    } else if (closingSale) {
        closingSection.classList.remove('hidden');
        document.getElementById('pop-close-property-id').textContent = closingSale.propertyId;
        document.getElementById('pop-close-value').textContent = formatCurrency(closingSale.salePrice);
        document.getElementById('pop-close-room').textContent = "โอนกรรมสิทธิ์เมื่อ: " + formatThaiDate(closingSale.transferDate);
        
        document.getElementById('pop-close-agent-lbl-1').textContent = "ผู้เกี่ยวข้องรับแบ่งส่วนค่าคอม";
        let staffNames = [];
        closingSale.stakeholders.forEach(sh => {
            const sObj = state.staff.find(st => st.id === sh.staffId);
            if (sObj) staffNames.push(`${sObj.name} (${formatCurrency(sh.commission)})`);
        });
        document.getElementById('pop-close-agent-val-1').textContent = staffNames.join(', ');
        
        document.getElementById('pop-close-agent-lbl-2').textContent = "รายได้รวมของบริษัท (3%)";
        document.getElementById('pop-close-agent-val-2').textContent = formatCurrency(closingSale.companyRevenue);

        if (closingSale.evidenceBase64) {
            evidenceImg.src = closingSale.evidenceBase64;
            evidenceImg.classList.remove('hidden');
            noEvidence.classList.add('hidden');
        } else {
            evidenceImg.classList.add('hidden');
            noEvidence.classList.remove('hidden');
        }
    } else {
        closingSection.classList.add('hidden');
    }

    // Show popup
    document.getElementById('customer-popup-modal').classList.remove('hidden');
    lucide.createIcons();
}

function triggerClientEdit() {
    if (!activePopupCaseId) return;
    document.getElementById('customer-popup-modal').classList.add('hidden');
    switchTab('cases');
    loadCaseIntoForm(activePopupCaseId);
}

/* ==========================================================================
   EXPORT PDF REPORT GENERATORS
   ========================================================================== */

// 1. Export Global Screen print
function handleGlobalPrint() {
    const element = document.getElementById('print-area');
    
    // Choose active panel tab context name for filename
    const activePanel = document.querySelector('.tab-panel.active');
    const tabName = activePanel ? activePanel.id.replace('tab-', '') : 'report';
    
    const opt = {
        margin:       [10, 10, 10, 10],
        filename:     `lalcondo-${tabName}-${getTodayString()}.pdf`,
        image:        { type: 'jpeg', quality: 0.98 },
        html2canvas:  { scale: 2, useCORS: true, backgroundColor: '#ffffff' },
        jsPDF:        { unit: 'mm', format: 'a4', orientation: 'landscape' }
    };
    
    // Alert info
    alert("ระบบกำลังสร้างไฟล์รายงาน PDF ในรูปแบบแนวนอนสำหรับพิมพ์รายงาน กรุณารอสักครู่...");
    
    html2pdf().set(opt).from(element).save();
}

// 2. Export individual customer invoice
function handleClientInvoicePrint() {
    if (!activePopupCaseId) return;
    
    const element = document.querySelector('.modal-container');
    const c = state.cases.find(item => item.id === activePopupCaseId);
    const clientName = c ? c.name.replace(/\s+/g, '_') : 'client';
    
    const opt = {
        margin:       [15, 15, 15, 15],
        filename:     `lalcondo-invoice-${clientName}-${getTodayString()}.pdf`,
        image:        { type: 'jpeg', quality: 0.98 },
        html2canvas:  { scale: 2, useCORS: true, backgroundColor: '#ffffff' },
        jsPDF:        { unit: 'mm', format: 'a4', orientation: 'portrait' }
    };
    
    alert("ระบบกำลังทำการพิมพ์ข้อมูลรายละเอียดเคสและเงินปันผลลูกค้าในแนวตั้ง...");
    
    html2pdf().set(opt).from(element).save();
}
