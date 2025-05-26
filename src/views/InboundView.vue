<template>
    <div class="container my-4">
      <h1>Halaman Inbound</h1>
  
      <div class="d-flex justify-content-end mb-3">
        <button class="btn btn-success" @click="() => showModal = true">+ New Inbound</button>
      </div>
  
      <StuffTable
        :data="inbound"
        :tableTh="['No', 'Nama Barang', 'Total Penambahan', 'Bukti Foto', 'Aksi']"
        :tableTd="['stuff.name', 'total', 'proof_file']"
        :actionBtn="['detail', 'delete']"
        :itemDetail="['proof_file']"
        title="Daftar Penambahan Stok"
        :exportBtn="['export-excel']"
        @export-excel="exportExcel"
      />
  
      <ModalForm modalTitle="Tambah Stok Barang" :isModalOpen="showModal" @close-modal="() => showModal = false">
        <form @submit.prevent="handleSubmitForm">
          <!-- form fields -->
        </form>
      </ModalForm>
    </div>
  </template>
  
  <script>
  import StuffTable from "../components/Table.vue";
  import ModalForm from "@/components/ModalForm.vue";
  import axios from "axios";
  import * as XLSX from "xlsx";
  import { API_BASE_URL } from "../constant.js";
  
  export default {
    name: "InboundView",
    components: { StuffTable, ModalForm },
    data() {
      return {
        inbound: [],
        showModal: false,
        stuffs: [],
        payload: {
          stuff_id: "",
          total: 0,
          proof_file: null,
        },
      };
    },
    methods: {
      fetchdata() {
        axios
          .get(API_BASE_URL + "/inbound-stuffs", {
            headers: {
              Authorization: "Bearer " + localStorage.getItem("access_token"),
            },
          })
          .then((res) => {
            // map item.proof_file menjadi URL lengkap
            this.inbound = res.data.data.map((item) => ({
              ...item,
              proof_file: item.proof_file
                ? API_BASE_URL + "/storage/" + item.proof_file
                : null,
            }));
          })
          .catch((err) => {
            if (err.response?.status === 401) {
              localStorage.removeItem("user");
              localStorage.removeItem("access_token");
              this.$router.replace("/");
            }
            console.error("Gagal ambil data:", err);
          });
      },
  
      fetchstuffs() {
        axios
          .get(API_BASE_URL + "/stuffs", {
            headers: {
              Authorization: "Bearer " + localStorage.getItem("access_token"),
            },
          })
          .then((res) => {
            this.stuffs = res.data.data;
          })
          .catch((err) => {
            // error handling
          });
      },
      handleSubmitForm() {
        if (!this.payload.stuff_id || !this.payload.total || !this.payload.proof_file) {
          alert("Semua kolom wajib diisi!");
          return;
        }
  
        const formData = new FormData();
        formData.append("stuff_id", this.payload.stuff_id);
        formData.append("total", this.payload.total);
        formData.append("proof_file", this.payload.proof_file);
  
        axios
          .post(API_BASE_URL + "/inbound-stuffs", formData, {
            headers: {
              Authorization: "Bearer " + localStorage.getItem("access_token"),
              "Content-Type": "multipart/form-data",
            },
          })
          .then(() => {
            this.message = "Data berhasil ditambahkan!";
            this.success = true;
            this.showModal = false;
            this.fetchdata();
            this.resetForm();
          })
          .catch((err) => {
            console.error("Gagal submit data:", err);
            this.message = "Gagal menyimpan data!";
            this.success = false;
          });
      },
  
      resetForm() {
        this.payload = {
          stuff_id: "",
          total: 0,
          proof_file: null,
        };
        const fileInput = document.getElementById("proof_file");
        if (fileInput) fileInput.value = "";
      },
  
      exportExcel() {
        const ws = XLSX.utils.json_to_sheet(
          this.inbound.map((item, index) => ({
            No: index + 1,
            NamaBarang: item.stuff?.name || "-",
            TotalItem: item.total,
            BuktiFoto: item.proof_file,
            TglPemasukanBarang: new Date(item.created_at).toLocaleDateString("id-ID", {
              dateStyle: "long",
            }),
          }))
        );
  
        const wb = XLSX.utils.book_new();
        XLSX.utils.book_append_sheet(wb, ws, "Daftar Penambahan Stok");
        XLSX.writeFile(wb, "daftar_penambahan_barang.xlsx");
      },
    },
  
    mounted() {
    this.fetchdata();
    this.fetchstuffs();
  },
};
  </script>
  