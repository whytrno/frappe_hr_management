<template>
	<div class="flex flex-col bg-white rounded w-full py-6 px-4 border-none">
		<h2 class="text-lg font-bold text-gray-900">Hey, {{ employee?.data?.nama_lengkap }} 👋</h2>

		<div class="font-medium text-sm text-gray-500 mt-1.5" v-if="lastLog">
			Last {{ lastLogType }} was at {{ lastLogTime }}
		</div>
		<Button
			v-if="!hasCheckedOutToday"
			class="mt-4 mb-1 drop-shadow-sm py-5 text-base"
			id="open-checkin-modal"
			@click="handleEmployeeCheckin"
		>
			<template #prefix>
				<FeatherIcon
					:name="nextAction.action === 'In' ? 'arrow-right-circle' : 'arrow-left-circle'"
					class="w-4"
				/>
			</template>
			{{ nextAction.label }}
		</Button>
	</div>

	<ion-modal
		ref="modal"
		trigger="open-checkin-modal"
		:initial-breakpoint="1"
		:breakpoints="[0, 1]"
	>
		<div class="h-120 w-full flex flex-col items-center justify-center gap-5 p-4 mb-5">
			<div class="flex flex-col gap-1.5 mt-2 items-center justify-center">
				<div class="font-bold text-xl">
					{{ dayjs(checkinTimestamp).format("hh:mm:ss a") }}
				</div>
				<div class="font-medium text-gray-500 text-sm">
					{{ dayjs().format("D MMM, YYYY") }}
				</div>
			</div>

			<span v-if="locationStatus" class="font-medium text-gray-500 text-sm">
				{{ locationStatus }}
			</span>

			<div class="rounded border-4 translate-z-0 block overflow-hidden w-full h-170">
				<div id="map" class="h-full w-full"></div>
				<div id="map22" class="h-full w-full"></div>
			</div>

			<div class="rounded border-4 translate-z-0 block overflow-hidden w-full h-170">
				<iframe
					width="100%"
					height="170"
					frameborder="0"
					scrolling="no"
					marginheight="0"
					marginwidth="0"
					style="border: 0"
					:src="`https://maps.google.com/maps?q=${latitude},${longitude}&hl=en&z=15&amp;output=embed`"
				>
				</iframe>
			</div>

			<Button variant="solid" class="w-full py-5 text-sm" @click="submitLog(nextAction.action)">
				Confirm {{ nextAction.label }}
			</Button>
		</div>
	</ion-modal>
</template>

<script setup>
import { createResource, createListResource, toast, FeatherIcon } from "frappe-ui"
import { computed, inject, ref, onMounted, onBeforeUnmount, nextTick } from "vue"
import { IonModal, modalController } from "@ionic/vue"
import { getAllLokasiSite } from "@/data/lokasi_site"
import L from "leaflet";  // Import leaflet
import "leaflet/dist/leaflet.css";

const DOCTYPE = "Absensi"

const socket = inject("$socket")
const employee = inject("$employee")
const dayjs = inject("$dayjs")
const checkinTimestamp = ref(null)
const latitude = ref(0)
const longitude = ref(0)
const locationStatus = ref("")
const lokasiSite = ref(null)
let map = null;  // Global map instance

const settings = createResource({
	url: "hr_management.api.get_hr_settings",
	auto: true,
})



// const checkins = createListResource({
// 	doctype: DOCTYPE,
// 	fields: ["name", "employee", "employee_name", "tipe", "time", "device_id"],
// 	filters: {
// 		employee: employee.data.name,
// 	},
// 	orderBy: "time desc",
// })
const checkins = createListResource({
	doctype: DOCTYPE,
	fields: ["*"],
	filters: {
		karyawan: employee.data.nrp,
	},
	orderBy: "waktu_absen desc",
})

checkins.reload()

const lastLog = computed(() => {
	if (checkins.list.loading || !checkins.data) return {}
	return checkins.data[0]
})

const lastLogType = computed(() => {
	return lastLog?.value?.tipe === "In" ? "check-in" : "check-out"
})

const nextAction = computed(() => {
	return lastLog?.value?.tipe === "In"
		? { action: "Out", label: "Check Out" }
		: { action: "In", label: "Check In" }
})

const lastLogTime = computed(() => {
	const timestamp = lastLog?.value?.waktu_absen
	const formattedTime = dayjs(timestamp).format("hh:mm a")

	if (dayjs(timestamp).isToday()) return formattedTime
	else if (dayjs(timestamp).isYesterday()) return `${formattedTime} yesterday`
	else if (dayjs(timestamp).isSame(dayjs(), "year"))
		return `${formattedTime} on ${dayjs(timestamp).format("D MMM")}`

	return `${formattedTime} on ${dayjs(timestamp).format("D MMM, YYYY")}`
})

const hasCheckedOutToday = computed(() => {
  // Cek apakah log terakhir adalah check-out dan terjadi pada hari ini
  const lastLogTimestamp = lastLog?.value?.waktu_absen;
  const isToday = dayjs(lastLogTimestamp).isToday();
  const isCheckOut = lastLog?.value?.tipe === "Out";

  // Jika log terakhir adalah check-out dan hari ini, return true
  return isToday && isCheckOut;
});


function handleLocationSuccess(position) {
	latitude.value = position.coords.latitude
	longitude.value = position.coords.longitude

	locationStatus.value = `
		Latitude: ${Number(latitude.value).toFixed(5)}°,
		Longitude: ${Number(longitude.value).toFixed(5)}°
	`
}

function handleLocationError(error) {
	locationStatus.value = "Unable to retrieve your location"
	if (error) locationStatus.value += `: ERROR(${error.code}): ${error.message}`
}

const fetchLocation = () => {
	if (!navigator.geolocation) {
		locationStatus.value = "Geolocation is not supported by your current browser"
	} else {
		console.log("Locating...")
		navigator.geolocation.getCurrentPosition(handleLocationSuccess, handleLocationError)
	}
}

const handleEmployeeCheckin = () => {
	checkinTimestamp.value = dayjs().format("YYYY-MM-DD HH:mm:ss")

	fetchLocation()

	// nextTick(() => {
	if (!map) {
		map = L.map("map22").setView([latitude.value, longitude.value], 15);  // Set initial view to the current location

		// Tambah tile layer dari OpenStreetMap
		L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
			maxZoom: 19,
			attribution: '© OpenStreetMap'
		}).addTo(map);

		// Tambah marker di titik tengah
		L.marker([latitude.value, longitude.value]).addTo(map);

		// Tambah lingkaran dengan radius 25 meter dari titik tengah
		L.circle([latitude.value, longitude.value], {
			color: "blue",
			fillColor: "#76a5af",
			fillOpacity: 0.3,
			radius: 25,
		}).addTo(map);
	}
//   });
}

const submitLog = (logType) => {
	const action = logType === "In" ? "Check-in" : "Check-out"

	checkins.insert.submit(
		{
			employee: employee.data.name,
			tipe: logType,
			time: checkinTimestamp.value,
			latitude: latitude.value,
			longitude: longitude.value,
		},
		{
			onSuccess() {
				modalController.dismiss()
				toast({
					title: "Success",
					text: `${action} successful!`,
					icon: "check-circle",
					position: "bottom-center",
					iconClasses: "text-green-500",
				})
			},
			onError() {
				toast({
					title: "Error",
					text: `${action} failed!`,
					icon: "alert-circle",
					position: "bottom-center",
					iconClasses: "text-red-500",
				})
			},
		}
	)
}

onMounted(() => {
	socket.emit("doctype_subscribe", DOCTYPE)
	socket.on("list_update", (data) => {
		if (data.doctype == DOCTYPE) {
			checkins.reload()
		}
	})
})

onBeforeUnmount(() => {
	socket.emit("doctype_unsubscribe", DOCTYPE)
	socket.off("list_update")
})
</script>
