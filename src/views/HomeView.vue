<template>
  <main>
    <div class="container">
      <div class="row">
        <div class="col-5">
          <div class="card mb-4" @drop="onDrop" @dragover="onDragOver">
            <div class="card-body">
              <input id="paperinput" v-model="paperinput" type="text" class="form-control" placeholder="Paper DOI...">
              <button class="btn btn-primary mt-2" @click="fetchGenerate">Add</button>
              <button class="btn btn-primary ms-2 mt-2" @click="clearAll">Clear all</button>
            </div>
          </div>
          <div>
            <div id="inputAccordion" class="accordion mb-4">
              <div v-for="paper in inputpapers" :key="paper.id" class="accordion-item">
                <h2 class="accordion-header">
                  <button :id="'button-' + paper.oa_id" class="accordion-button collapsed" type="button"
                    data-bs-toggle="collapse" :data-bs-target="'#' + paper.oa_id" aria-expanded="false"
                    aria-controls="collapseOne">
                    {{ paper.title }}
                  </button>
                </h2>
                <div :id="paper.oa_id" class="accordion-collapse collapse" data-bs-parent="#inputAccordion">
                  <div class="accordion-body">
                    {{ paper.id }}
                  </div>
                </div>
              </div>
            </div>
          </div>
          <div>
            <div id="paperAccordion" class="accordion">
              <div v-for="paper in outputpapers" :key="paper.id" class="accordion-item">
                <h2 class="accordion-header">
                  <button :id="'button-' + paper.oa_id" class="accordion-button collapsed" type="button"
                    data-bs-toggle="collapse" :data-bs-target="'#' + paper.oa_id" aria-expanded="false"
                    aria-controls="collapseOne">
                    {{ paper.title }}
                  </button>
                </h2>
                <div :id="paper.oa_id" class="accordion-collapse collapse" data-bs-parent="#paperAccordion">
                  <div class="accordion-body">
                    {{ paper.id }}
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
        <div class="col-7">
          <div v-show="isLoading" class="spinner-border" role="status">
            <span class="visually-hidden">Loading...</span>
          </div>
          <div v-show="inputs.length > 0">
            <label for="yearrange" class="form-label">Min Year</label>: {{ year }}
            <input id="yearrange" v-model="year" type="range" :min="min_year" :max="max_year" class="form-range"
              @change="generateGraph">
            <label for="citedrange" class="form-label">Min cited by count</label>: {{ citedby }}
            <input id="citedrange" v-model="citedby" type="range" :min="min_citedby" :max="max_citedby"
              class="form-range" @change="generateGraph">
            <div id="visnetwork"></div>
          </div>
        </div>
      </div>
    </div>
  </main>
</template>

<script>
import { Network } from "vis-network"
import { DataSet } from "vis-data"
import { pipeline } from '@huggingface/transformers'

import * as pdfjsLib from 'pdfjs-dist'

// Set the worker source to the file in the public directory
pdfjsLib.GlobalWorkerOptions.workerSrc = '/pdf.worker.min.mjs'



export default {
  data: () => ({
    isLoading: false,
    network: null,
    paperinput: null,
    papers: new Map(),
    connections: [],
    inputs: [],
    year: 0,
    citedby: 0
  }),
  computed: {
    paperKeys() {
      return Array.from(this.papers.keys())
    },
    filteredPapers() {
      return Array.from(this.papers.values()).filter(
        (p) => (p.publication_year >= this.year && p.cited_by_count >= this.citedby) || this.inputs.includes(p.id)
      ).sort((a, b) => b.cited_by_count - a.cited_by_count)
    },
    inputpapers() {
      return Array.from(this.papers.values()).filter((p) => this.inputs.includes(p.id))
    },
    outputpapers() {
      return this.filteredPapers.filter((p) => !this.inputs.includes(p.id))
    },
    min_year() {
      return Math.min(...Array.from(this.papers.values()).map((p) => p.publication_year))
    },
    max_year() {
      return Math.max(...Array.from(this.papers.values()).map((p) => p.publication_year))
    },
    min_citedby() {
      return 0
    },
    max_citedby() {
      return 50
    }
  },
  methods: {
    onDrop(e) {
      e.preventDefault()
      console.log('y', e, e.dataTransfer?.items, e.dataTransfer?.files)
      const files = e.dataTransfer.files
      if (files.length > 0 && files[0].type === 'application/pdf') {
        this.readPdf(e.dataTransfer.files[0])
      }
    },
    onDragOver(e) {
      e.preventDefault()
    },
    readPdf(file) {
      const fileReader = new FileReader();
      fileReader.onload = async () => {
        const typedarray = new Uint8Array(fileReader.result);
        const pdf = await pdfjsLib.getDocument(typedarray).promise;
        const textContent = [];

        for (let pageNum = 1; pageNum <= pdf.numPages; pageNum++) {
          const page = await pdf.getPage(pageNum);
          const textContentPage = await page.getTextContent();
          textContent.push(...textContentPage.items.map(item => item.str));
        }

        console.log('PDF Text:', textContent.join('\n'));
        console.log(this)
        this.runExtraction(textContent.join('\n'))
      };
      fileReader.readAsArrayBuffer(file);
    },
    async runExtraction(text) {
      text
    },
    clearAll() {
      this.papers = new Map()
      this.connections = []
      this.inputs = []
    },
    async fetchGenerate() {
      this.isLoading = true

      const papers = await this.fetchPaper(this.paperinput, true)
      this.paperinput = ''
      this.inputs.push(papers[0].id)
      console.log('fetched initial paper', papers.length)

      await this.fetchCitations(papers[0])
      this.year = Array.from(this.papers.values())
        .map((p) => p.publication_year)
        .sort((a, b) => b - a).at(50) || this.min_year
      this.citedby = this.min_citedby
      Array.from(this.papers.values()).forEach((p) => {
        p.oa_id = p.id.split('/').pop()
      })

      this.generateGraph()
    },
    async generateGraph() {
      this.isLoading = true
      if (this.network) this.network.destroy()


      const nodesDs = new DataSet(
        this.filteredPapers.map((p) => {
          let color = `hsl(230, 70%, ${90 - (p.publication_year - 1970)}%)`
          if (this.inputs.includes(p.id)) {
            color = 'black'
          }
          return {
            id: p.id,
            title: p.title,
            color: color,
            value: p.cited_by_count,
            size: p.cited_by_count
          }
        })
      )

      const edges = new DataSet(this.connections.filter(
        (c) => this.paperKeys.includes(c.from) && this.paperKeys.includes(c.to))
      )

      const container = document.getElementById("visnetwork")
      const data = {
        nodes: nodesDs,
        edges: edges,
      }

      this.isLoading = false
      this.network = new Network(container, data, {
        layout: {
          improvedLayout: false
        },
        nodes: {
          shape: 'dot',
          shadow: true,
          scaling: { max: 50 },
        }
      })
      this.network.on('click', (props) => {
        if (props.nodes.length > 0) {
          console.log(props.nodes)
          const el = document.getElementById('button-' + props.nodes[0].split('/').pop())
          el.click()
          el.scrollIntoView()
        }
      })
    },

    async fetchCitations(paper) {
      let result = await fetch(paper.cited_by_api_url)
      const papers = (await result.json()).results
      console.log('got citations', papers.length)
      for (const citPaper of papers) {
        this.papers.set(citPaper.id, citPaper)
        this.connections.push({ from: citPaper.id, to: paper.id })

        const ids = citPaper.referenced_works.map((w) => w.split('/').pop()).slice(0, 70).join('|')
        result = await fetch(`https://api.openalex.org/works?filter=ids.openalex:${ids}`)
        if (!result.ok) continue

        const references = (await result.json()).results
        references.forEach((r) => {
          console.log('add ref')
          this.papers.set(r.id, r)
          if (r.id !== paper.id) {
            this.connections.push({ from: citPaper.id, to: r.id })
          }
          r.referenced_works.forEach((r_in) => {
            this.connections.push({ from: r.id, to: r_in.id })
          })
        })
      }

      return papers
    },

    async fetchPaper(id, withReferences = false) {
      let result = await fetch(`https://api.openalex.org/works/${this.paperinput}`)

      const paper = await result.json()
      this.papers.set(paper.id, paper)

      let references = []
      if (withReferences) {
        const ids = paper.referenced_works.map((w) => w.split('/').pop()).slice(0, 70).join('|')
        result = await fetch(`https://api.openalex.org/works?filter=ids.openalex:${ids}`)

        references = (await result.json()).results
        references.forEach((r) => {
          this.papers.set(r.id, r)
          this.connections.push({ from: paper.id, to: r.id })
        })
      }
      return [paper, ...references]
    }
  }
}
</script>

<style scoped>
#paperAccordion {
  max-height: 60vh;
  overflow-y: scroll;
}

#inputAccordion {
  max-height: 20vh;
  overflow-y: scroll;
}

#visnetwork {
  height: 60vh;
}
</style>
<style>
.vis-network {
  overflow: visible !important;
}
</style>
