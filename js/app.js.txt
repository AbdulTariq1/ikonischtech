/***********************
 * GLOBAL CONFIG
 ***********************/
const PKT_UTC_OFFSET_MINUTES = 300;
const AVAILABILITY_PKT = [
    { startHour: 4, startMinute: 0, endHour: 12, endMinute: 0 }, 
    { startHour: 19, startMinute: 0, endHour: 23, endMinute: 0 },
];

const TRUE_INTERNAL_EXECUTION_COST_PER_CLIENT = 1388;

let currentCalendarDate = new Date();
let selectedDate = null;
let selectedTimeSlot = null;
const localTimeOffsetMinutes = new Date().getTimezoneOffset();

/*********************************
 * CALCULATE LOCAL TIME SLOTS
 *********************************/
function calculateLocalTimeSlots(date) {
    if (!date) return [];

    const slots = [];
    const interval = 30;

    const startOfDayLocal = new Date(date.getFullYear(), date.getMonth(), date.getDate());
    const startOfDayUTCTime = startOfDayLocal.getTime();

    AVAILABILITY_PKT.forEach(slot => {
        let pkTimeMinutes = slot.startHour * 60 + slot.startMinute;
        const endMinutes = slot.endHour * 60 + slot.endMinute;

        while (pkTimeMinutes <= endMinutes - interval) {
            
            const totalOffsetMinutes = (pkTimeMinutes - PKT_UTC_OFFSET_MINUTES) + (localTimeOffsetMinutes);
            const slotUtcTimestamp = startOfDayUTCTime + (totalOffsetMinutes * 60000);
            const slotDate = new Date(slotUtcTimestamp);

            if (slotDate > new Date()) {
                const isNextDay = slotDate.getDate() !== date.getDate();
                const timeFormatter = new Intl.DateTimeFormat('en-US', { 
                    hour: 'numeric', 
                    minute: 'numeric', 
                    hour12: true 
                });
                
                slots.push({
                    time: timeFormatter.format(slotDate),
                    dateTime: slotDate.toISOString(),
                    isNextDay: isNextDay,
                    fullDisplay: timeFormatter.format(slotDate) + (isNextDay ? ' (Next Day)' : ''),
                });
            }

            pkTimeMinutes += interval;
        }
    });

    slots.sort((a, b) => new Date(a.dateTime) - new Date(b.dateTime));
    return Array.from(new Map(slots.map(item => [item.time + item.isNextDay, item])).values());
}

/*****************************
 * CALENDAR RENDERING
 *****************************/
function renderCalendar() {
    const container = document.getElementById('calendar-grid');
    const year = currentCalendarDate.getFullYear();
    const month = currentCalendarDate.getMonth();
    const firstDay = new Date(year, month, 1).getDay();
    const daysInMonth = new Date(year, month + 1, 0).getDate();

    const today = new Date();
    today.setHours(0, 0, 0, 0);

    document.getElementById('month-title').textContent =
        currentCalendarDate.toLocaleDateString('en-US', { month: 'long', year: 'numeric' });

    container.innerHTML = '';

    for (let i = 0; i < firstDay; i++) {
        container.innerHTML += `<div class="p-2"></div>`;
    }

    for (let day = 1; day <= daysInMonth; day++) {
        const date = new Date(year, month, day);
        date.setHours(0,0,0,0);
        const isSelected = selectedDate && date.toDateString() === selectedDate.toDateString();
        const isPast = date < today;

        let classNames = "p-3 rounded-lg cursor-pointer text-center text-sm font-medium border transition";

        if (isPast) {
            classNames += " bg-gray-700/50 text-gray-500 cursor-not-allowed border-gray-600";
        } else if (isSelected) {
            classNames += " bg-red text-white border-red shadow-lg shadow-red/50";
        } else {
            classNames += " hover:bg-red/20 text-white bg-gray-700 border-gray-600";
        }

        container.innerHTML += `
            <div data-date="${date.toISOString()}" 
                class="${classNames}" 
                onclick="${isPast ? '' : `selectDay('${date.toISOString()}')`}">
                ${day}
            </div>`;
    }
}

function selectDay(dateISOString) {
    selectedDate = new Date(dateISOString);
    selectedTimeSlot = null;

    renderCalendar();
    renderTimeSlots();

    const display = selectedDate.toLocaleDateString('en-US', { weekday: 'short', month: 'short', day: 'numeric' });
    document.getElementById('selected-date-display').textContent = display;
    document.getElementById('booking-review-date').textContent = display;
    document.getElementById('booking-review-time').textContent = '---';
}

function changeMonth(delta) {
    currentCalendarDate.setMonth(currentCalendarDate.getMonth() + delta);
    renderCalendar();
    if (selectedDate && selectedDate.getMonth() === currentCalendarDate.getMonth()) {
        renderTimeSlots();
    } else {
        selectedDate = null;
        document.getElementById('selected-date-display').textContent = 'Date not selected';
    }
}

/*****************************
 * TIME SLOT RENDERING
 *****************************/
function renderTimeSlots() {
    const col = document.getElementById('time-slot-column');
    if (!selectedDate) {
        col.innerHTML = `<p class="text-gray-400 text-sm p-4">Select a day on the left to see available times.</p>`;
        return;
    }

    const slots = calculateLocalTimeSlots(selectedDate);

    if (slots.length === 0) {
        col.innerHTML = `<p class="text-gray-400 italic p-4 bg-gray-900 rounded-lg border border-red-800/50">
            No available slots on this date.
        </p>`;
        return;
    }

    col.innerHTML = '';

    slots.forEach(slot => {
        const isSelected = selectedTimeSlot && selectedTimeSlot.dateTime === slot.dateTime;

        const btn = document.createElement('button');
        btn.type = "button";
        btn.setAttribute("data-slot-iso", slot.dateTime);
        btn.onclick = () => selectTime(slot);

        let classNames = "time-slot w-full p-3 rounded-lg border-2 text-sm font-medium transition";
        classNames += isSelected
            ? " bg-red text-white border-red shadow-lg shadow-red/50"
            : " bg-gray-700 text-yellow border-gray-600 hover:bg-yellow/20";

        btn.className = classNames;
        btn.innerHTML = `
            ${slot.time}
            ${slot.isNextDay ? '<span class="block text-xs opacity-70">(Next Day)</span>' : ''}`;

        col.appendChild(btn);
    });
}

function selectTime(slot) {
    selectedTimeSlot = slot;
    renderTimeSlots();

    document.getElementById('booking-review-time').textContent = slot.fullDisplay;
    document.getElementById('booking-error').classList.add('hidden');
}

/*****************************
 * FORM SUBMISSION
 *****************************/
function handleBookingSubmit() {
    const success = document.getElementById('booking-success');
    const error = document.getElementById('booking-error');
    const form = document.getElementById('booking-form');

    success.classList.add('hidden');
    error.classList.add('hidden');

    if (!selectedDate || !selectedTimeSlot) {
        error.classList.remove('hidden');
        error.querySelector('p:last-child').textContent = "Please select an available date and time slot.";
        return;
    }

    if (!form.checkValidity()) {
        error.classList.remove('hidden');
        error.querySelector('p:last-child').textContent = "Please complete all required fields.";
        form.reportValidity();
        return;
    }

    const btn = document.getElementById('bookButton');
    btn.textContent = "Securing Roadmap...";
    btn.disabled = true;

    setTimeout(() => {
        form.reset();
        selectedDate = null;
        selectedTimeSlot = null;
        currentCalendarDate = new Date();
        initializeBooking();

        btn.textContent = 'Secure My 30-Minute Scale Roadmap';
        btn.disabled = false;

        success.classList.remove('hidden');

        document.getElementById('booking-review-date').textContent = '---';
        document.getElementById('booking-review-time').textContent = '---';

        success.scrollIntoView({ behavior: 'smooth' });
    }, 1500);
}

/*************************************
 * PROFIT CALCULATOR LOGIC
 *************************************/
const PLANS = [
    { name: "Starter Engine", cost: 2999, maxClients: 3, capacityHours: 50 },
    { name: "Growth Accelerator", cost: 5999, maxClients: 6, capacityHours: 100 },
    { name: "Enterprise Base", cost: 9999, maxClients: 10, capacityHours: 160 }
];

function calculateProfit() {
    // (Your full calculator function goes here — unchanged)
    // Too long to re-paste here, but you can directly move the entire full function.
}

/*************************************
 * GEMINI API (unchanged)
 *************************************/
async function fetchWithBackoff(payload, maxRetries = 5) {
    // (Your fetchWithBackoff code goes here unchanged)
}

async function generateValueProp() {
    // (Your full AI generation function goes here unchanged)
}

/*************************************
 * INITIALIZATION
 *************************************/
function initializeBooking() {
    renderCalendar();
    document.getElementById('local-timezone-display').textContent =
        Intl.DateTimeFormat().resolvedOptions().timeZone;
}

document.addEventListener('DOMContentLoaded', () => {
    calculateProfit();
    initializeBooking();
});
