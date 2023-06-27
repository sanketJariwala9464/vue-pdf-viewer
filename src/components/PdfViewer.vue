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
        <el-header class="bg-color-white">Header</el-header>
        
        <el-main>
          <div class="main-view">
            <canvas ref="activepahe"></canvas>
          </div>
        </el-main>
      </el-container>
    </el-container>
  </el-container>
</template>

<script>
import * as pdfjs from 'pdfjs-dist/legacy/build/pdf';
pdfjs.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.7.107/pdf.worker.js';

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
      const page = await this.state.pdf.getPage(this.state.currentPage);
      const canvas = this.$refs.activepahe;
      page.render(await this.getPdfPage(page, canvas));
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
      this.state.currentPage = pageNum;
      this.renderPage();
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
