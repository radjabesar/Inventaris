<template>
  <div class="container py-4">
    <h2>Kelola Barang</h2>

    <!-- Form Tambah/Edit Barang -->
    <form @submit.prevent="submitForm" class="mb-4">
      <div class="mb-2">
        <label>Nama Barang:</label>
        <input v-model="form.name" class="form-control" required />
      </div>

      <div class="mb-2">
        <label>Tipe Barang:</label>
        <select v-model="form.type" class="form-control" required>
          <option value="" disabled>Pilih tipe</option>
          <option value="Lab">Lab</option>
          <option value="HTL/KLN">HTL/KLN</option>
          <option value="Sarpras">Sarpras</option>
        </select>
      </div>

      <div class="mb-2">
        <label>Foto Barang:</label>
        <input
          type="file"
          accept="image/*"
          class="form-control"
          @change="e => form.image = e.target.files[0]"
          :required="!isEdit"
        />
        <small v-if="isEdit && form.imageUrl" class="text-muted">
          (Biarkan kosong kalau tidak ingin mengganti foto)
        </small>
      </div>

      <button class="btn btn-primary">
        {{ isEdit ? 'Update' : 'Tambah' }}
      </button>
    </form>

    <!-- Pesan sukses / error -->
    <div v-if="message" class="alert" :class="success ? 'alert-success' : 'alert-danger'">
      {{ message }}
    </div>

    <!-- Cari -->
    <input
      v-model="search"
      placeholder="Cari nama atau ID..."
      class="form-control my-3"
    />

    <!-- Tabel Barang -->
    <StuffTable
      :data="filteredStuffs"
      :tableTh="tableTh"
      :tableTd="tableTd"
      :actionBtn="actionBtn"
      :itemDetail="itemDetail"
      title="Daftar Barang"
      :exportBtn="['export-pdf', 'export-excel']"
      @edit-item="handleEdit"
      @delete-item="deleteItem"
      @export-pdf="printPDF"
      @export-excel="exportExcel"
    />
  </div>
</template>

<script>
import StuffTable from "../components/Table.vue";
import axios from "axios";
import { API_BASE_URL } from "../constant.js";
import jsPDF from "jspdf";
import * as XLSX from "xlsx";

export default {
  name: "StuffView",
  components: { StuffTable },
  data() {
    return {
      stuffs: [],
      form: { id: null, name: "", type: "", image: null, imageUrl: "" },
      isEdit: false,
      search: "",
      message: "",
      success: false,

      // Tambahkan kolom Gambar
      tableTh: ["No", "Gambar", "Nama", "Tipe", "Aksi"],
      // Kolom gambar kita map ke prop "imageUrl"
      tableTd: ["imageUrl", "name", "type"],
      actionBtn: ["edit", "delete", "detail"],
      // Di detail modal, tampilkan juga gambar
      itemDetail: ["imageUrl", "name", "type"],
    };
  },
  computed: {
    filteredStuffs() {
      const s = this.search.toLowerCase();
      return this.stuffs.filter(
        (item) =>
          item.name.toLowerCase().includes(s) ||
          String(item.id).includes(s)
      );
    },
  },
  methods: {
    getAuthHeaders() {
      return { Authorization: "Bearer " + localStorage.getItem("access_token") };
    },

    handleAuthError(err) {
      if (err.response?.status === 401) {
        localStorage.removeItem("user");
        localStorage.removeItem("access_token");
        this.$router.replace("/");
      }
    },

    fetchStuffs() {
      axios
        .get(`${API_BASE_URL}/stuffs`, { headers: this.getAuthHeaders() })
        .then((res) => {
          this.stuffs = res.data.data.map((item) => ({
            ...item,
            imageUrl: item.image
              ? `${API_BASE_URL}/storage/${item.image}`
              : null,
          }));
        })
        .catch((err) => {
          this.handleAuthError(err);
          console.error("Gagal ambil data:", err);
          this.message = "Gagal memproses data!";
          this.success = false;
        });
    },

    submitForm() {
      let config = { headers: this.getAuthHeaders() };
      let url = `${API_BASE_URL}/stuffs`;
      let method = "post";
      let payload = this.form;

      // Jika ada file (baru atau edit dengan ganti foto), pakai FormData
      if (this.form.image) {
        const fd = new FormData();
        fd.append("name", this.form.name);
        fd.append("type", this.form.type);
        fd.append("image", this.form.image);
        payload = fd;
        config.headers["Content-Type"] = "multipart/form-data";
      }

      if (this.isEdit) {
        url = `${API_BASE_URL}/stuffs/${this.form.id}`;
        method = this.form.image ? "post" : "patch";
        // Jika update dengan FormData, Laravel butuh _method=PATCH
        if (this.form.image) fd.append("_method", "PATCH");
      }

      axios[method](url, payload, config)
        .then(() => {
          this.success = true;
          this.message = this.isEdit
            ? "Berhasil mengubah data!"
            : "Berhasil menambahkan data!";
          this.resetForm();
          this.fetchStuffs();
        })
        .catch((err) => {
          this.handleAuthError(err);
          console.error("Gagal kirim data:", err);
          this.message = this.isEdit
            ? "Gagal mengubah barang!"
            : "Gagal menambahkan barang!";
          this.success = false;
        });
    },

    handleEdit(item) {
      this.form = {
        id: item.id,
        name: item.name,
        type: item.type,
        image: null,           // kosongkan file input
        imageUrl: item.imageUrl, // tampilkan preview lama
      };
      this.isEdit = true;
    },

    deleteItem(id) {
      axios
        .delete(`${API_BASE_URL}/stuffs/${id}`, {
          headers: this.getAuthHeaders(),
        })
        .then(() => {
          this.success = true;
          this.message = "Berhasil menghapus data!";
          this.fetchStuffs();
        })
        .catch((err) => {
          this.handleAuthError(err);
          console.error("Gagal hapus:", err);
          this.message = "Gagal menghapus data!";
          this.success = false;
        });
    },

    resetForm() {
      this.form = { id: null, name: "", type: "", image: null, imageUrl: "" };
      this.isEdit = false;
      // reset file input DOM jika perlu
      const inp = this.$el.querySelector('input[type="file"]');
      if (inp) inp.value = "";
    },

    printPDF() {
      const doc = new jsPDF();
      doc.setFontSize(18);
      doc.text("Daftar Barang", 20, 20);
      let y = 30;
      doc.setFontSize(12);
      doc.text("No", 20, y);
      doc.text("Nama", 40, y);
      doc.text("Tipe", 100, y);
      y += 10;
      this.stuffs.forEach((item, i) => {
        doc.text((i + 1).toString(), 20, y);
        doc.text(item.name, 40, y);
        doc.text(item.type, 100, y);
        y += 10;
      });
      doc.save("daftar_barang.pdf");
    },

    exportExcel() {
      const ws = XLSX.utils.json_to_sheet(
        this.stuffs.map((item, i) => ({
          No: i + 1,
          Nama: item.name,
          Tipe: item.type,
        }))
      );
      const wb = XLSX.utils.book_new();
      XLSX.utils.book_append_sheet(wb, ws, "Daftar Barang");
      XLSX.writeFile(wb, "daftar_barang.xlsx");
    },
  },

  mounted() {
    this.fetchStuffs();
  },
};
</script>
