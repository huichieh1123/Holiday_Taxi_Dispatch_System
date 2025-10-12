<template>
  <div>
    <div v-if="detailLoading" class="message">Loading details...</div>
    <div v-if="selectedBooking" class="details">
      <h2>Booking Details: {{ selectedBooking.__ref }}</h2>

      <div class="detail-grid">
        <div><strong>Leg:</strong> {{ selectedBooking.__leg }}</div>
        <div><strong>Flight No:</strong> {{ selectedBooking.__flightNo || 'N/A' }}</div>
        <div><strong>Arrival:</strong> {{ formatDT(selectedBooking.__arrivalDate) || 'N/A' }}</div>
        <div><strong>Departure:</strong> {{ formatDT(selectedBooking.__departureDate) || 'N/A' }}</div>
        <div><strong>Passenger:</strong> {{ selectedBooking.__passenger || 'N/A' }}</div>
      </div>

      <h3 class="mt-2">Raw JSON</h3>
      <pre>{{ JSON.stringify(selectedBooking, null, 2) }}</pre>

      <!-- Location Link Generator -->
      <div class="link-generator">
        <h4>Generate Location Update Link</h4>
        <button @click="generateLocationLink">Generate Link</button>
        <div v-if="locationUpdateLink" class="generated-link">
          <p>Share this link with the driver:</p>
          <input type="text" :value="locationUpdateLink" readonly />
          <button @click="copyLinkToClipboard">Copy</button>
        </div>
      </div>

      <!-- Update Driver Form -->
      <div class="update-form">
        <h3>Update Driver & Vehicle</h3>
        <div class="form-grid">
          <div class="form-group">
            <label>Driver Name:</label>
            <input v-model="driverForm.name" placeholder="e.g., John Doe" />
          </div>
          <div class="form-group">
            <label>License Number:</label>
            <input v-model="driverForm.licenseNumber" placeholder="e.g., 123456" />
          </div>
          <div class="form-group">
            <label>Driver Phone:</label>
            <input v-model="driverForm.phoneNumber" placeholder="e.g., +441234567890" />
          </div>

          <div class="form-group full-width">
            <label>Available Contact Methods:</label>
            <div class="checkbox-group">
              <div v-for="option in contactOptions" :key="option.name" class="checkbox-item">
                <input type="checkbox" :id="`opt-${option.name}`" v-model="option.checked" />
                <label :for="`opt-${option.name}`">{{ option.name }}</label>
              </div>
            </div>
          </div>

          <div class="form-group">
            <label>Preferred Contact Method:</label>
            <select v-model="preferredContactMethod">
              <option v-for="method in availableContactMethods" :key="method" :value="method">{{ method }}</option>
            </select>
          </div>

          <div class="form-group">
            <label>Vehicle Brand:</label>
            <input v-model="vehicleForm.brand" placeholder="e.g., Toyota" />
          </div>
          <div class="form-group">
            <label>Vehicle Model:</label>
            <input v-model="vehicleForm.model" placeholder="e.g., Prius" />
          </div>
          <div class="form-group">
            <label>Vehicle Color:</label>
            <input v-model="vehicleForm.color" placeholder="e.g., Blue" />
          </div>
          <div class="form-group">
            <label>Vehicle Registration:</label>
            <input v-model="vehicleForm.registration" placeholder="e.g., AB-123-XYZ" />
          </div>
          <div class="form-group full-width">
            <label>Vehicle Description:</label>
            <input v-model="vehicleForm.description" placeholder="e.g., Branded with 'Acme Transfers'" />
          </div>
        </div>
        <button @click="updateDriver" :disabled="updatingDriver || !selectedBooking">
          {{ updatingDriver ? 'Updating...' : 'Update Driver' }}
        </button>
        <div v-if="driverUpdateMessage" class="message success">{{ driverUpdateMessage }}</div>
        <div v-if="driverUpdateError" class="message error">{{ driverUpdateError }}</div>
      </div>

    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, watch, computed } from 'vue';
import type { BookingNorm, DriverForm, VehicleForm } from '../../types/booking';

const props = defineProps<{
  selectedBooking: BookingNorm | null;
  detailLoading: boolean;
}>();

// --- State Management ---
const driverForm = ref({ name: '', phoneNumber: '', licenseNumber: '' });
const vehicleForm = ref<VehicleForm>({ brand: '', model: '', color: '', description: "", registration: '' });

const contactOptions = ref([
  { name: 'VOICE', checked: true },
  { name: 'SMS', checked: true },
  { name: 'WHATSAPP', checked: true }
]);
const preferredContactMethod = ref('VOICE');

const availableContactMethods = computed(() => {
  return contactOptions.value.filter(opt => opt.checked).map(opt => opt.name);
});

watch(availableContactMethods, (newMethods) => {
  if (!newMethods.includes(preferredContactMethod.value)) {
    preferredContactMethod.value = newMethods.length > 0 ? newMethods[0] : '';
  }
});

const updatingDriver = ref(false);
const driverUpdateMessage = ref<string | null>(null);
const driverUpdateError = ref<string | null>(null);

const locationUpdateLink = ref('');
const API_BASE_URL = import.meta.env.VITE_API_BASE_URL;

// --- Lifecycle & Watchers ---
watch(() => props.selectedBooking, () => {
  driverUpdateMessage.value = null;
  driverUpdateError.value = null;
  locationUpdateLink.value = '';
});

// --- Helper Functions ---
const formatDT = (s?: string | null) => s ? new Date(s).toLocaleString() : '';

// --- Driver Update Logic ---
const updateDriver = async () => {
  if (!props.selectedBooking?.__ref) return;

  const linkWasGenerated = locationUpdateLink.value !== ''; // Check before update

  updatingDriver.value = true;
  driverUpdateMessage.value = null;
  driverUpdateError.value = null;
  try {
    const ref = props.selectedBooking.__ref;
    const payload = {
      driver: {
        name: driverForm.value.name,
        phoneNumber: driverForm.value.phoneNumber,
        licenseNumber: driverForm.value.licenseNumber,
        preferredContactMethod: preferredContactMethod.value,
        contactMethods: availableContactMethods.value,
      },
      vehicle: { ...vehicleForm.value },
    };

    const res = await fetch(`${API_BASE_URL}/api/bookings/${encodeURIComponent(ref)}/driver`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload),
    });

    if (!res.ok) {
      const data = await res.json().catch(() => ({}));
      if (data.reason === 'TOO_MANY_DISTINCT_VEHICLE_IDENTIFIERS_FOR_THIS_BOOKING') {
        const idToDelete = prompt('Too many vehicles assigned. Please enter the ID of the OLD vehicle to delete and retry:');
        if (idToDelete?.trim()) {
          await deleteVehicleAndRetryDriverUpdate(ref, idToDelete.trim());
          return; // Exit after starting the retry flow
        }
        throw new Error('Vehicle deletion cancelled by user.');
      }
      if (data.message && typeof data.message === 'object' && data.message.errors) {
        throw new Error(JSON.stringify(data.message.errors, null, 2));
      }
      throw new Error(data.message?.message || data.detail || 'Update failed');
    }

    let successMsg = 'Driver & vehicle updated successfully.';
    if (linkWasGenerated) {
      generateLocationLink(); // Auto-regenerate link
      successMsg += ' The location link has been automatically updated.';
    }
    driverUpdateMessage.value = successMsg;

  } catch (e: any) {
    driverUpdateError.value = e.message;
  }
  finally {
    updatingDriver.value = false;
  }
};

const deleteVehicleAndRetryDriverUpdate = async (bookingRef: string, vehicleId: string) => {
  driverUpdateMessage.value = `Attempting to delete old vehicle ${vehicleId}...`;
  driverUpdateError.value = null;
  updatingDriver.value = true;
  try {
    const res = await fetch(`${API_BASE_URL}/api/bookings/${encodeURIComponent(bookingRef)}/vehicles/${encodeURIComponent(vehicleId)}`, { method: 'DELETE' });
    if (!res.ok) {
      const data = await res.json().catch(() => ({}));
      throw new Error(data.message || 'Failed to delete old vehicle.');
    }
    driverUpdateMessage.value = 'Old vehicle deleted. Retrying driver update...';
    setTimeout(() => updateDriver(), 1500);
  } catch (e: any) {
    driverUpdateError.value = e.message;
    updatingDriver.value = false;
  }
};

// --- Link Generation ---
const generateLocationLink = () => {
  if (props.selectedBooking?.__ref) {
    const vehicleId = vehicleForm.value.registration;
    if (!vehicleId) {
      alert('Please enter a vehicle registration number first.');
      return;
    }
    locationUpdateLink.value = `${window.location.origin}/update-location/${props.selectedBooking.__ref}/${vehicleId}`;
  }
};

const copyLinkToClipboard = () => {
  if (locationUpdateLink.value) {
    navigator.clipboard.writeText(locationUpdateLink.value).then(() => alert('Link copied!'), () => alert('Failed to copy.'));
  }
};
</script>

<style scoped>
/* All original styles */
.message { padding: 15px; border-radius: 4px; margin: 20px 0; }
.error { background-color: #f8d7da; color: #721c24; border: 1px solid #f5c6cb; }
.success { background-color: #d4edda; color: #155724; border: 1px solid #c3e6cb; }
.details { margin-top: 30px; padding: 20px; border: 1px solid #ccc; border-radius: 8px; background-color: #fafafa; }
.detail-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 8px 20px; margin-bottom: 14px; }
pre { background-color: #eee; padding: 15px; border-radius: 4px; white-space: pre-wrap; word-wrap: break-word; max-height: 400px; overflow-y: auto; }
.update-form { margin-top: 20px; }
.update-form h3 { margin-bottom: 15px; }
.form-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 15px; margin-bottom: 20px; }
.form-group { display: flex; flex-direction: column; }
.form-group label { margin-bottom: 5px; font-weight: bold; color: #333; }
.form-group input, .form-group select { padding: 10px; border: 1px solid #ccc; border-radius: 4px; }
.form-group.full-width { grid-column: 1 / -1; }
.checkbox-group { display: flex; gap: 15px; align-items: center; }
.checkbox-item { display: flex; align-items: center; gap: 5px; }
.mt-2 { margin-top: 0.5rem; }
.link-generator { margin-top: 20px; padding: 15px; border: 1px solid #eee; border-radius: 4px; }
.generated-link { margin-top: 15px; }
.generated-link p { margin-bottom: 5px; }
.generated-link input { width: calc(100% - 80px); padding: 8px; border: 1px solid #ccc; border-radius: 4px; margin-right: 10px; }
button { padding: 10px 20px; border: none; background-color: #007bff; color: white; border-radius: 4px; cursor: pointer; transition: background-color 0.3s; }
button:disabled { background-color: #ccc; cursor: not-allowed; }
button:hover:not(:disabled) { background-color: #0056b3; }
@media (max-width: 768px) {
  .generated-link { display: flex; flex-direction: column; }
  .generated-link input { width: 100%; margin-right: 0; margin-bottom: 10px; }
  .generated-link button { width: 100%; }
}
</style>
