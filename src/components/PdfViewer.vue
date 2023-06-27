<template>
  <el-container style="height: 100vh; border: 1px solid #eee">
    <el-container>
      <el-aside width="200px">
        <div class="side-pdf-view" :class="state.currentPage === pageNumber ? 'active-pdf' : ''" v-for="pageNumber in state.totalPages" :key="pageNumber">
          <div class="pdf-number" :class="state.currentPage === pageNumber ? 'pdf-number-active' : ''">{{ pageNumber }}</div>
          <div class="pdf-otheroption">
            <el-dropdown size="mini" trigger="click">
              <button class="el-button download-button el-button--danger el-button--mini is-plain el-dropdown-selfdefine">
                <i class="el-icon-more"></i>
              </button>
              <el-dropdown-menu slot="dropdown">
                <el-dropdown-item>Action 1</el-dropdown-item>
              </el-dropdown-menu>
            </el-dropdown>
          </div>
          <canvas :ref="'canvas' + pageNumber" :id="'pdf-page-' + pageNumber" @click="openPage(pageNumber)" class="side-pdf-canvas"></canvas>
        </div>
      </el-aside>

      <el-container>
        <el-header class="bg-color-white">
          <el-popover
            placement="bottom"
            width="500"
            trigger="hover"
          >
            <div>
              <div class="block">
                <span class="demonstration">Storke Width</span>
                <el-slider v-model="settings.pen.width" :max="50"></el-slider>
              </div>
              <div class="block">
                <span class="demonstration">Storke Opacity</span>
                <el-slider v-model="settings.pen.opacity"></el-slider>
              </div>
              <div class="block">
                <div class="demonstration">Color</div>
                <el-color-picker v-model="settings.pen.color"></el-color-picker>
              </div>
            </div>
            <el-button slot="reference" :type="!mode.ispen ? 'danger' : 'success'" @click="setPenMode()" class="download-button" size="small"><i class="el-icon-edit"></i></el-button>
          </el-popover>
          <!-- <button @click="setAddTextMode()" class="el-button download-button el-button--danger el-button--small is-plain el-dropdown-selfdefine">
            <img src="../../public/text.png" alt="">
          </button> -->
        </el-header>
        
        <el-main>
          <!-- {{ mode }} -->
          <div class="main-view">
            <div id="PDF-stage" class="d-view-pdf"></div>
          </div>
        </el-main>
      </el-container>
    </el-container>
  </el-container>
</template>

<script>
import * as pdfjs from 'pdfjs-dist/legacy/build/pdf';
pdfjs.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.7.107/pdf.worker.js';
import Konva from 'konva';

export default {
  name: 'PdfViewer',
  props: {
    src: {
      type: String,
      required: true,
    },
  },
  data() {
    return {
      state: {
        pdf: null,
        totalPages: 0,
        currentPage: 1,
        scale: 1,
        layer: {},
        layerimage: {},
        stage: {},
        text: {},
      },
      mode: {
        ispen: false,
        isaddText: false,
      },
      settings: {
        pen: {
          color: '#df4b26',
          width: 5,
          opacity: 100,
        },
      }
    };
  },
  mounted() {
    console.log(this.src)
    this.loadPdf();
  },
  methods: {
    async loadPdf() {
      const loadingTask = pdfjs.getDocument(this.src);
      this.state.pdf = await loadingTask.promise;
      this.state.totalPages = this.state.pdf.numPages;
      if (this.state.totalPages > 0) {
        this.showpdfPgaeInSideBar()
      }
    },
    async showpdfPgaeInSideBar() {
      this.renderPage();
      for (let pageNumber = 1; pageNumber <= this.state.totalPages; pageNumber++) {
        const page = await this.state.pdf.getPage(pageNumber);
        const canvas = this.$refs['canvas' + pageNumber][0];
        page.render(await this.getPdfPage(page, canvas, true));
      }
    },
    async renderPage() {
      const self = this;
      const page = await this.state.pdf.getPage(this.state.currentPage);
      const canvas = document.createElement('canvas');
      const renderContext = await this.getPdfPage(page, canvas);
      const image = new Image();
      image.src = await page.render(renderContext).promise.then(() => {
        return canvas.toDataURL();
      });
      image.onload = function () {
        self.stage = new Konva.Stage({
          container: 'PDF-stage',
          width: renderContext.viewport.width,
          height: renderContext.viewport.height,
          id: 'pdf-stage',
        });

        self.layer = new Konva.Layer({
          draggable: false,
        });
        self.stage.add(self.layer);
        self.layerimage = new Konva.Image();
        self.text = new Konva.Text();
        self.layer.add(self.layerimage);
        self.layerimage.setImage(image);  
        self.layer.batchDraw();
        self.initPen();
        self.initText();
      };
    },
    async getPdfPage(page, canvas, issidebar = false) {
      const context = canvas.getContext('2d');
      const viewport = page.getViewport({ scale: this.state.scale });

      canvas.width = viewport.width;
      canvas.height = viewport.height;

      if (issidebar) {
        canvas.style.width ='100%';
        canvas.style.height='100%';
      } else {
        canvas.style.width ='60%';
        canvas.style.height='60%';
      }

      context.clearRect(0, 0, canvas.width, canvas.height); // Clear the canvas

      const renderContext = {
        canvasContext: context,
        viewport: viewport,
      };

      return renderContext;
    },
    async openPage(pageNum) {
      this.mode.ispen = false;
      this.mode.isaddText = false;
      this.state.currentPage = pageNum;
      this.renderPage();
      this.setCurson();
    },
    async setPenMode() {
      this.mode.ispen = !this.mode.ispen;
      if (this.mode.ispen) {
        this.setCurson('pen');
      } else {
        this.setCurson();
      }
    },
    async setAddTextMode() {
      this.mode.isaddText = !this.mode.isaddText;
      if (this.mode.isaddText) {
        this.setCurson('text');
      } else {
        this.setCurson();
      }
    },
    setCurson(type = '') {
      let cursor = '';
      switch (type) {
        case 'pen':
          cursor = 'url(../../pencile.cur), auto';
        break;
        case 'text':
          cursor = 'text';
        break;
      }
      const element = document.getElementById('PDF-stage');
      if (cursor !== '' && type !== '') {
        element.style.cursor = cursor;
      } else {
        element.style.cursor = 'default';
      }
    },
    initPen() {
      let isDrawing = false;
      const self = this;
      self.stage.on('mousedown touchstart', function () {
        console.log('mousedown ')
        if (self.mode.ispen) {
          isDrawing = true;
          const pos = self.stage.getPointerPosition();
          self.lastLine = new Konva.Line({
            stroke: self.settings.pen.color,
            strokeWidth: self.settings.pen.width,
            opacity: self.settings.pen.opacity / 100,
            globalCompositeOperation: 'source-over',
            lineCap: 'round',
            lineJoin: 'round',
            points: [pos.x, pos.y, pos.x, pos.y],
            draggable: true,
          });
          self.layer.add(self.lastLine);
        }
      });

      self.stage.on('mouseup touchend', function () {
        isDrawing = false;
      });

      self.stage.on('mousemove touchmove', function (e) {
        if (!self.mode.ispen) return;
        if (!isDrawing) return;
        e.evt.preventDefault();
        const pos = self.stage.getPointerPosition();
        const newPoints = self.lastLine.points().concat([pos.x, pos.y]);
        self.lastLine.points(newPoints);
      });
    },
    initText() {
      const self = this;
      self.stage.on("click tap", function () {
        if (self.mode.isaddText) {
          const pos = self.stage.getPointerPosition();
          self.text.setAttrs({
            x: pos.x,
            y: pos.y,
            width: 100,
            height: 20,
            fontSize: 20,
            text: 'Some text here',
            draggable: true,
            id: 'text',
          });
          self.layer.add(self.text);

          self.setAddTextMode();
        }
      });
    }
  },
};
</script>

<style scoped>
.main-view {
  margin: 10px;
  display: flex;
  justify-content: center;
  align-items: center;
}

.side-pdf-view {
  margin: 10px;
  position: relative;
  background-color: #fff;
  border-radius: 4px;
  box-shadow: 0 2px 12px 0 rgb(0 0 0 / 10%);
}

.active-pdf {
  border: 2px solid #ef5b5b;
}

.active-icon {
  font-size: 20px;
  color: #0dea1f;
}

.pdf-otheroption {
  opacity: 1;
  -webkit-transition: .3s;
  transition: .3s;
  position: absolute;
  top: 0;
  right: 0;
  padding: 2px;
  background-color: transparent;
}

.side-pdf-canvas {
  cursor: pointer;
}

.bg-color-white {
  background-color: #d9d9d9;
  display: flex;
  justify-content: center;
  align-items: center;
}

.pdf-number {
  position: absolute;
  background: #545c64;
  padding: 2px 5px;
  color: #fafafa;
  border-bottom-right-radius: 3px;
  width: 25px;
  height: 26px;
  display: flex;
  justify-content: center;
  top: 0;
  align-items: center;
  left: 0;
}

.pdf-number-active {
  background-color: #ef5b5b;
}

.download-button {
  padding-left: 10px;
  padding-right: 10px;
}
</style>
