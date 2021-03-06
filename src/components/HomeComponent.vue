<template>
  <v-layout align-start>
    <v-flex>
      <v-data-table
        :headers="headers"
        :items="items"
        sort-by="location"
        class="elevation-1"
        :items-per-page="50"
        :footer-props="{ 'items-per-page-options': [50, 100, 250, 500] }"
      >
        <template v-slot:top>
          <v-toolbar flat color="white">
            <v-toolbar-title>Conteo de Inventario</v-toolbar-title>
            <v-divider class="mx-4" inset vertical></v-divider>
            <v-btn
              :loading="loadingUploadBtn"
              :disabled="disableUploadBtn"
              color="green"
              class="ma-2 white--text"
              @click="showUploadModal"
            >
              Cargar
              <v-icon right dark> mdi-cloud-upload </v-icon>
            </v-btn>
            <v-btn
              :loading="loadingDownloadBtn"
              :disabled="disableDownloadBtn"
              color="red"
              class="ma-2 white--text"
              @click="downloadString"
            >
              Descargar
              <v-icon right dark> mdi-cloud-download </v-icon>
            </v-btn>
            <v-spacer></v-spacer>
            <v-text-field
              class="text-xs-center"
              v-model="sku"
              append-icon="mdi-barcode"
              label="Paquete"
              single-line
              hide-details
              v-on:keyup="validateKeyPressed"
            ></v-text-field>
            <v-spacer></v-spacer>
            <v-chip class="ma-2" color="green" outlined>
              Ubicación : 
              <b>{{ location }}</b>
            </v-chip>
          </v-toolbar>
        </template>
        <template v-slot:[`item.actions`]="{ item }">
          <!-- <template v-slot:item.options="{ item }"> -->
          <v-icon size="sm" color="red" class="mr-1" @click="deleteItem(item)"
            >mdi-delete</v-icon
          >
        </template>
      </v-data-table>
    </v-flex>
    <v-dialog v-model="uploadModal" max-width="500px">
      <v-card>
        <v-card-text>
          <template>
            <v-file-input
              v-model="file"
              accept="file/*.csv"
              label="Cargar Plantilla"
            ></v-file-input>
            <v-btn @click="UploadInventoryItems()" color="blue" dark>
              <v-icon left>mdi-cloud-upload</v-icon>Cargar
            </v-btn>
            <v-btn @click="hideUploadModal()" color="blue darken-1" text
              >Cancelar</v-btn
            >
          </template>
        </v-card-text>
      </v-card>
    </v-dialog>
  </v-layout>
</template>

<script>
import Papa from "papaparse";
import InventoryItemModel from "../models/InventoryItemModel";

export default {
  components: {},
  data: () => ({
    headers: [
      { text: "Tracking", sortable: true, value: "sku" },
      { text: "Ubicación", sortable: true, value: "location" },
      { text: "Opciones", value: "options", sortable: false },
    ],
    items: [],
    sku: "",
    location: "",

    inventoryItemModel: new InventoryItemModel(),
    loadingUploadBtn: false,
    disableUploadBtn: true,
    loadingDownloadBtn: false,
    disableDownloadBtn: true,
    fileProcessed: null,
    file: null,
    uploadModal: false,
  }),
  watch: {},
  mounted() {},

  methods: {
    // displayNotification(type, message) {
    //   this.$swal.fire({
    //     position: "top-end",
    //     type: type,
    //     title: message,
    //     showConfirmButton: false,
    //     timer: 1000,
    //   });
    // },
    validateKeyPressed(e) {
      if (e.keyCode === 13) {
        this.addSku();
      }
    },

    UploadInventoryItems() {
      if (this.file == null) {
        alert("El archivo es de un formato incorrecto o no se ha cargado.");
        // this.displayNotification(
        //   "error",
        //   "El archivo es de un formato incorrecto o no se ha cargado."
        // );
        return;
      }

      let me = this;

      Papa.parse(this.file, {
        header: true,
        complete: function (results) {
          me.fileProcessed = results.data;
          var inventoryItemList = results.data.map((a) => {
            return {
              sku: a.SKU,
              location: a.LOCATION,
            };
          });

          inventoryItemList = inventoryItemList.filter(
            (x) =>
              x.sku != undefined &&
              x.sku != "" &&
              x.location != undefined &&
              x.location != ""
          );

          if (inventoryItemList.length == 0) {
            alert(
              "No se pudo procesar el archivo, revise el formato o los datos."
            );
            //me.displayNotification("error", "No se pudo procesar el archivo, revise el formato.");
            return;
          }

          me.uploadModal = false;
          alert("Se cargaron los registros correctamente.");

          inventoryItemList.forEach((a) => {
            me.items.push({ sku: a.sku, location: a.location });
          });
        },
      });
    },

    showUploadModal() {
      this.uploadModal = true;
    },
    hideUploadModal() {
      this.uploadModal = false;
      this.file = null;
    },

    addSku() {
      let firstLetter = this.sku.charAt(0);

      if (
        this.items.length == 0 &&
        firstLetter != "." &&
        this.location === ""
      ) {
        alert("Específique ubicación");
        //this.displayNotification("error", "Especifique Ubicación.");
        this.cleanSku();
        return;
      }

      if (this.items.length == 0 && firstLetter === ".") {
        this.location = this.sku;
        this.cleanSku();
        return;
      }

      if (this.items.length > 0 && firstLetter === ".") {
        this.location = this.sku;
        this.cleanSku();
        return;
      }

      this.items.push({ sku: this.sku, location: this.location });

      if (this.disableDownloadBtn) this.disableDownloadBtn = false;

      this.cleanSku();
    },
    cleanSku() {
      this.sku = "";
    },

    downloadString() {
      let text = this.getInventoryItemFormatString();
      let fileType = "text";
      let fileName = "archivoinventario.txt";

      var blob = new Blob([text], { type: fileType });
      console.log(text);
      var a = document.createElement("a");
      a.download = fileName;
      a.href = URL.createObjectURL(blob);
      a.dataset.downloadurl = [fileType, a.download, a.href].join(":");
      a.style.display = "none";
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      setTimeout(function () {
        URL.revokeObjectURL(a.href);
      }, 1500);
    },

    getInventoryItemFormatString() {
      let result = "";

      let locations = [...new Set(this.items.map(x => x.location))];

      locations.forEach((x) => {
        result = result + x + "\n";

        let skus = this.items.filter(y => 
          x == y.location
        ).map( (a) => {return a.sku});

        skus.forEach((z) => {
          result = result + z + "\n";
        });
      });

      console.log(result);
      return result;
    },
  },
};
</script>
