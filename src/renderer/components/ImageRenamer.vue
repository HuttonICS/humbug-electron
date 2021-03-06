<template>
  <div>
    <b-navbar toggleable="md" type="dark" variant="primary" class="navbar-margin">
      <b-navbar-brand :title="$t('navigationBack')" href="#" @click="back"><back-icon :size="48" /> <img id="logo" src="~@/assets/logo-no-bg.svg" width="30" height="30" class="d-inline-block align-top"> {{ $t('menuFileBulkImageRenaming') }}</b-navbar-brand>
    </b-navbar>
    <b-container>
      <p>{{ $t('imageRenamerMessage') }}</p>

      <b-form>
        <b-row>
          <b-col sm=4>
            <b-form-group :label="$t('labelSourceFolder')" label-for="sourcePath">
              <b-input-group>
                <b-form-input id="sourcePath" type=text :value="sourcePath" :placeholder="$t('labelSourceFolder')" @change.native="updateSourcePath"></b-form-input>
                <b-input-group-append>
                  <b-btn variant="secondary" @click="selectSource()">...</b-btn>
                </b-input-group-append>
              </b-input-group>
            </b-form-group>
          </b-col>
          <b-col sm=4>
            <b-form-group :label="$t('labelTargetFolder')" label-for="targetPath">
              <b-input-group>
                <b-form-input id="targetPath" type=text :value="targetPath" :placeholder="$t('labelTargetFolder')" @change.native="updateTargetPath"></b-form-input>
                <b-input-group-append>
                  <b-btn variant="secondary" @click="selectTarget()">...</b-btn>
                </b-input-group-append>
              </b-input-group>
            </b-form-group>
          </b-col>
          <b-col sm=4>
            <b-form-group :label="$t('labelOnMissingBarcode')" label-for="onMissing">
              <b-form-select id="onMissing" :options="onMissingOptions" :value="onMissingBarcode" @change.native="updateOnMissingBarcode"></b-form-select>
            </b-form-group>
          </b-col>
        </b-row>
      </b-form>
      <b-btn @click="start()" variant="primary">{{ $t('buttonStart') }}</b-btn>
      <!-- Show the progress -->
      <b-progress :value="index" :max="images.length" show-progress :animated="index < images.length" :variant="index < images.length ? 'info' : 'success'" class="img-progress" v-if="images && images.length > 0"/>

      <!-- This image is used to render the base64 image and get the barcode. It's hidden because we don't need to show it -->
      <img ref="img" :src="currentImage" class="img-responsive w-100" @load="onImageLoad()" style="display: none;"/>

      <p />

      <b-table striped hover bordered responsive :fields="fields" :items="processedItems" v-if="processedItems && processedItems.length > 0" >
        <template slot="status" slot-scope="data">
          <template v-if="data.value">
            <alert-icon />
            {{ data.value }}
          </template>
          <check-icon v-else />
        </template>
      </b-table>
    </b-container>
  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import AlertIcon from 'vue-material-design-icons/Alert.vue'
import BackIcon from 'vue-material-design-icons/ArrowLeft.vue'
import CheckIcon from 'vue-material-design-icons/Check.vue'
const { BrowserBarcodeReader, BrowserQRCodeReader } = require('@zxing/library/esm5')

const fs = require('fs')
const path = require('path')
const app = require('electron').remote
const dialog = app.dialog

export default {
  data: function () {
    return {
      onMissingOptions: [this.$t('valueOnMisingBarcodeSkipImage'), this.$t('valueOnMisingBarcodeCopyOriginal')],
      images: [],
      fields: ['from', 'to', 'status'],
      processedItems: [],
      index: 0,
      currentImage: null,
      currentImagePath: null
    }
  },
  components: {
    AlertIcon, BackIcon, CheckIcon
  },
  computed: {
    ...mapGetters([
      'sourcePath',
      'targetPath',
      'onMissingBarcode'
    ])
  },
  methods: {
    // Navigate back to the landing page
    back: function () {
      this.$router.push('landing-page')
    },
    // Get the current image and set the base64 value to the <img>
    handleImage: function () {
      this.currentImagePath = path.join(this.sourcePath, this.images[this.index++])

      var base64 = this.getBase64(this.currentImagePath)
      this.currentImage = base64
    },
    updateSourcePath: function (e) {
      this.$store.dispatch('setSourcePath', e.target.value)
    },
    updateTargetPath: function (e) {
      this.$store.dispatch('setTargetPath', e.target.value)
    },
    updateOnMissingBarcode: function (e) {
      this.$store.dispatch('setOnMissingBarcode', e.target.value)
    },
    // Open dialog to choose the source folder
    selectSource () {
      var vm = this
      dialog.showOpenDialog({
        properties: ['openDirectory'],
        defaultPath: this.sourcePath
      }, function (file) {
        if (file) {
          vm.$store.dispatch('setSourcePath', file[0])
        }
      })
    },
    // Open dialog to choose the target folder
    selectTarget () {
      var vm = this
      dialog.showOpenDialog({
        properties: ['openDirectory'],
        defaultPath: this.targetPath ? this.targetPath : this.sourcePath
      }, function (file) {
        if (file) {
          vm.$store.dispatch('setTargetPath', file[0])
        }
      })
    },
    // When the image is loaded, try and get the barcode
    onImageLoad: function () {
      var vm = this
      this.getOneD()
        .then(result => vm.handleBarcode(result))
        .catch(err => {
          console.error(err)
          vm.getQr()
            .then(result => vm.handleBarcode(result))
            .catch(err => {
              console.error(err)
              vm.handleMissing()
            })
        })
    },
    // Try and read 1d barcodes
    getOneD: function () {
      return new BrowserBarcodeReader('video').decodeFromImage(this.$refs.img)
    },
    // Try and read 2d barcodes
    getQr: function () {
      return new BrowserQRCodeReader().decodeFromImage(this.$refs.img)
    },
    // Handle the case where no barcode is found
    handleMissing: function () {
      if (this.onMissingBarcode === this.$t('valueOnMisingBarcodeCopyOriginal')) {
        var fileName = path.basename(this.currentImagePath)
        fileName = fileName.replace(/\.[^/.]+$/, '')
        var ext = path.extname(this.currentImagePath)

        var targetFile = path.join(this.targetPath, fileName + ext)

        var counter = 1
        while (fs.existsSync(targetFile)) {
          targetFile = path.join(this.targetPath, fileName + ('-' + counter++) + ext)
        }

        fs.copyFileSync(this.currentImagePath, targetFile)

        this.processedItems.push({
          from: fileName,
          to: path.basename(targetFile),
          status: this.$t('valueOnMisingBarcodeCopyOriginal'),
          _cellVariants: {
            status: 'warning'
          }
        })
      } else {
        this.processedItems.push({
          from: path.basename(this.currentImagePath),
          status: this.$t('valueOnMisingBarcodeSkipImage'),
          _cellVariants: {
            status: 'warning'
          }
        })
      }
    },
    // Handle the case where a barcode is found
    handleBarcode: function (barcode) {
      // Copy image
      var ext = path.extname(this.currentImagePath)
      var targetFile = path.join(this.targetPath, barcode.text + ext)

      var counter = 1
      while (fs.existsSync(targetFile)) {
        targetFile = path.join(this.targetPath, barcode.text + ('-' + counter++) + ext)
      }

      fs.copyFileSync(this.currentImagePath, targetFile)

      this.processedItems.push({
        from: path.basename(this.currentImagePath),
        to: path.basename(targetFile),
        _cellVariants: {
          status: 'success'
        }
      })

      // Progress to the next image
      if (this.index < this.images.length) {
        this.handleImage()
      } else {
        this.currentImage = null
      }
    },
    // Start the process
    start: function () {
      var vm = this
      if (!this.sourcePath || !this.targetPath) {
        dialog.showMessageBox(app.getCurrentWindow(), {
          type: 'error',
          title: vm.$t('dialogErrorTitle'),
          message: vm.$t('dialogImageRenamingSpecifyFolders')
        })
        return
      }

      this.processedItems = []
      this.index = 0
      // Read source folder and get all images
      fs.readdir(this.sourcePath, function (error, list) {
        if (error) {
          console.error(error)
        }

        vm.images = list.filter(function (file) {
          const ext = path.extname(file).toLowerCase()
          return ext === '.jpg' || ext === '.jpeg' || ext === '.png'
        })

        // Start processing
        vm.handleImage()
      })
    }
  }
}
</script>

<style>
  .img-progress {
    margin-top: 15px;
  }
  .navbar-margin {
    margin-bottom: 15px;
  }
</style>
