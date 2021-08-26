<template>
  <div id="app">

    <div v-if="inError" id="error">
      <font-awesome-icon icon="exclamation-triangle"></font-awesome-icon> Sorry, something went wrong.
    </div>

    <div v-else>
    <figure>

    <div id="ol-wrapper" ref="olWrapper" ></div>

    <div id="caption-wrapper" ref="captionWrapper" v-if="validated && captionShown">
      <div id="caption-left-control" class="caption-control">
        <div v-if="!(currentScene < 0)" @click="changeScene('back')"><font-awesome-icon icon="arrow-left"></font-awesome-icon></div>
      </div>
      <div id="caption-center">
        <figcaption>
        <div v-html="captionCenter"></div>
        </figcaption>
        <div v-if="currentScene >= 0 && currentScene < screenplay.scenes.length" id="caption-bottom-control"><span class="counter">{{currentScene+1}}/{{screenplay.scenes.length}}</span> <font-awesome-icon class="control-bottom-icon" icon="eye-slash" @click="captionShown = false;"></font-awesome-icon> <font-awesome-icon class="control-bottom-icon" icon="external-link-alt" v-if="contextUrl.length > 0" @click="openContextUrl"></font-awesome-icon> </div>
        <div class="begin-callout" v-if="currentScene === -1" @click="changeScene(0)">Begin</div>

      </div>
      <div id="caption-right-control" class="caption-control">
          <div @click="changeScene('forward')"><font-awesome-icon :icon="currentScene >= screenplay.scenes.length ? 'undo-alt' : 'arrow-right'"></font-awesome-icon></div>
      </div>
    </div>
    <div id="caption-shower" v-if="!captionShown">
      <font-awesome-icon icon="eye" @click="captionShown=true;" class="control-bottom-icon"></font-awesome-icon>
    </div>

    </figure>

    </div>
    </div>
</template>

<script>



import { library as faLibrary, config as faConfig } from '@fortawesome/fontawesome-svg-core'
import { faExclamationTriangle, faArrowRight, faArrowLeft, faEye, faEyeSlash, faUndoAlt, faExternalLinkAlt  } from '@fortawesome/free-solid-svg-icons'
import { FontAwesomeIcon } from '@fortawesome/vue-fontawesome'

faConfig.autoAddCss = false; 
faLibrary.add(faExclamationTriangle, faArrowRight, faArrowLeft, faEye, faEyeSlash, faUndoAlt, faExternalLinkAlt);

import {Map, View} from 'ol';
import TileLayer from 'ol/layer/Tile';
import ImageLayer from 'ol/layer/Image';
import IIIF from 'ol/source/IIIF';
import IIIFInfo from 'ol/format/IIIFInfo';
import OSM from 'ol/source/OSM';
import Static from 'ol/source/ImageStatic';
import XYZ from 'ol/source/XYZ';
import TileJSON from 'ol/source/TileJSON';

import MarkdownIt from 'markdown-it';

const md = new MarkdownIt();

const imageMeasurer = (src) => {
  return new Promise((resolve, reject) => {
    let img = new Image()
    img.onload = () => resolve([img.height, img.width])
    img.onerror = reject
    img.src = src
  });
}

export default {
  name: 'PanelTruck',
  components: {
    'font-awesome-icon': FontAwesomeIcon
  },
  props: {
    'screenplaySrc': String,
  },
  data: function(){
    return {
      inError: false,
      validated: false,
      errorMessage: "",
      screenplay: {},
      currentScene: -1,
      layers: {},
      ol: null,
      captionShown: true,
      currentMaxExtent: []
    }
  },
  computed: {

    captionCenter: function(){
      if(this.currentScene < 0){
        let c = `<h1 class="title">${this.screenplay.metadata.title}</h1>`;
        if(this.screenplay.metadata.subtitle){ c += `<h2 class="subtitle">${this.screenplay.metadata.subtitle}</h2>` }
        if(this.screenplay.metadata.author){ c += `<h3 class="author">by ${this.screenplay.metadata.author}</h3>` }
        return c;
      }
      else if (this.currentScene >= this.screenplay.scenes.length) {
        let c = `<p><strong>${this.screenplay.metadata.title}</strong></p>`;
        if(this.screenplay.metadata.author){ c += `<p>by ${this.screenplay.metadata.author}</p>` }
        if(this.screenplay.metadata.publishedDate){ c += `<p>published on ${this.screenplay.metadata.publishedDate}</p>` }
        c += `<p class="small-credit">Created with panel-truck 0.2, <a href="https://leventhalmap.org">Leventhal Map and Education Center</a> at the Boston Public Library</p>`;
        return c;
      } else {
        let thisScene = this.screenplay.scenes[this.currentScene];
        let c = "";
        if(thisScene.caption.title && thisScene.caption.title.length > 0){
          c += `<h4 class="caption-title">${md.render(thisScene.caption.title)}</h4>`;
        }
        c += md.render(thisScene.caption.text);
        return c;
      }
     },

    contextUrl: function(){
      if(!this.validated || this.currentScene < 0 || this.curentScene >= this.screenplay.scenes.length ){
        return "";
      } else {
        return this.screenplay.scenes[this.currentScene].moreInfo ? this.screenplay.scenes[this.currentScene].moreInfo : this.screenplay.sources.filter(d=>{return d.id === this.screenplay.scenes[this.currentScene].source})[0].moreInfo ? this.screenplay.sources.filter(d=>{return d.id === this.screenplay.scenes[this.currentScene].source})[0].moreInfo : "";
      }
    }
  },
  methods: {
    validateScreenplay: function() {
      return new Promise((resolve, reject) => {
      fetch(this.screenplaySrc)
        .then(r => r.json())
        .then(r => {
          try {
          if(typeof r.metadata === "object" && Array.isArray(r.sources) && Array.isArray(r.scenes)) {
            this.inError = false;
            this.validated = true;
            this.screenplay = r;
            resolve();
          } else {
            reject();
          }
          }
          catch {
            reject();
          }
        })
        .catch(() => {
          reject();
        })
      });
    },

    changeScene: function(to, initial) {
      let nextSceneNumber;

      if(to === "forward") {
        nextSceneNumber = this.currentScene + 1;
      } else if (to === "back") {
        nextSceneNumber = this.currentScene - 1;
      } else if (typeof(to)==='number') {
        nextSceneNumber = to;
      } else {
        return;
      }

      if(nextSceneNumber < -1) {
        nextSceneNumber = -1;
      } else if( nextSceneNumber > this.screenplay.scenes.length ) {
        nextSceneNumber = -1;
      }

      let nextFunctionalScene;
      
      if(nextSceneNumber === -1) {
        nextFunctionalScene = 0;
      } else if(nextSceneNumber === this.screenplay.scenes.length) {
        nextFunctionalScene = this.screenplay.scenes.length - 1;
      } else {
        nextFunctionalScene = nextSceneNumber;
      }

      let currentSceneSource;

      if(this.currentScene === -1) {
        currentSceneSource = this.screenplay.scenes[0].source;
      } else if(this.currentScene === this.screenplay.scenes.length) {
        currentSceneSource = this.screenplay.scenes[this.currentScene - 1].source;
      } else {
        currentSceneSource = this.screenplay.scenes[this.currentScene].source;
      }
      
      this.currentScene = nextSceneNumber;

      if( initial || currentSceneSource != this.screenplay.scenes[nextFunctionalScene].source ) {
        this.changeSource(nextFunctionalScene)
          .then(() => { this.changeExtent(nextFunctionalScene) });
      } else {
        this.changeExtent(nextFunctionalScene, 3000);
      }


    },

    changeSource: function(sceneNumber) {
      return new Promise((resolve, reject) => {
      let newSource = this.screenplay.sources.filter(d=>{return d.id === this.screenplay.scenes[sceneNumber].source})[0];
      if(newSource.type.toLowerCase() === 'iiifmanifest') {

          let sequence = newSource.sequence ? newSource.sequence : 0;
          let canvas = newSource.canvas ? newSource.canvas : 0;
          let image = newSource.image ? newSource.image : 0;

          fetch(newSource.manifest)
              .then(r => r.json())
              .then(m => {
                  let imageEndpoint = m.sequences[sequence].canvases[canvas].images[image].resource.service['@id'];
                  this.loadNewIIIF(imageEndpoint)
                    .then(resolve)
                    .catch(reject);
              })
              .catch(reject);
      } else if(newSource.type.toLowerCase() === 'iiifimage') {
          this.loadNewIIIF(newSource.imageEndpoint)
                    .then(resolve)
                    .catch(reject);
      }
      else if(newSource.type.toLowerCase() === "staticimage") {
        this.loadNewImage(newSource.imageFile)
          .then(resolve)
          .catch(reject);
      } else if(newSource.type.toLowerCase() === "geomap"){
        this.loadNewGeoMap(newSource)
          .then(resolve)
          .catch(reject);
      } else {
        reject();
      }
      });
    },

    changeExtent: function(sceneNumber, transitionDuration) {

      let extent = this.screenplay.scenes[sceneNumber].extent ? this.screenplay.scenes[sceneNumber].extent : "fit";
      let bottomPadding = this.$refs.captionWrapper.clientHeight > 50 ? this.$refs.captionWrapper.clientHeight + 20 : 70;
      let fitOptions = { padding: [10, 10, bottomPadding, 10] };
      if(typeof(transitionDuration) === 'number') { fitOptions.duration = transitionDuration }

      if (extent === 'fit') {
            this.ol.getView().fit(this.currentMaxExtent, fitOptions);
			} else {
						this.ol.getView().fit(extent, fitOptions);
      }
    },

    loadNewIIIF: function(imageEndpoint) {
      return new Promise((resolve, reject) => {
        fetch(imageEndpoint)
            .then(r => r.json())
            .then(iiif => {
                var options = new IIIFInfo(iiif).getTileSourceOptions();
                if (options === undefined || options.version === undefined) {
                  return;
                }
                options.zDirection = -1;
                var iiifTileSource = new IIIF(options);
                this.currentMaxExtent = iiifTileSource.getTileGrid().getExtent();
                this.layers.iiif.setSource(iiifTileSource);
                this.layers.iiif.setVisible(true);
                this.layers.geoMap.setVisible(false);
                this.layers.staticImage.setVisible(false);
                resolve();
            })
          .catch(() => { this.inError = true; reject(); });
      });
    },

    loadNewImage: function(imageSrc) {
      return new Promise((resolve, reject) => {
      imageMeasurer(imageSrc)
        .then((hw)=> {
        this.layers.staticImage.setSource(
          new Static({
            url: imageSrc,
            imageExtent: [0, 0, hw[1], hw[0]]
          })
        );
        this.currentMaxExtent = [0, 0, hw[1], hw[0]];
        this.layers.staticImage.setVisible(true);
        this.layers.iiif.setVisible(false);
        this.layers.geoMap.setVisible(false);
        resolve();
    })
      .catch(()=>{ this.inError = true; reject(); });
      })
    },

    loadNewGeoMap: function(mapSrc) {
      return new Promise((resolve) => {
        if(mapSrc.tileXYZ) {
          this.layers.geoMap.setSource(new XYZ({ url: mapSrc.tileXYZ }));
        } else if(mapSrc.tileJSON) {
          this.layers.geoMap.setSource(new TileJSON({ url: mapSrc.tileJSON }));
        }
        else {
          this.layers.geoMap.setSource(new OSM());
        }
        this.currentMaxExtent = [-20026376.39,-20048966.10,20026376.39,20048966.10];
        this.layers.geoMap.setVisible(true);
        this.layers.staticImage.setVisible(false);
        this.layers.iiif.setVisible(false);
        resolve();
      });
    },

    openContextUrl: function() {
      window.open(this.contextUrl);
    }


    
  },
  mounted: function() {
    this.layers.iiif = new TileLayer({visible: false});
    this.layers.geoMap = new TileLayer({visible: false});
    this.layers.staticImage = new ImageLayer({visible: false});

    let wrapperDom = this.$refs.olWrapper;
  
    // can't initialized OL until the DOM is mounted and sized
    // there must be a more elegant way to do this but this works for now
    let checker = setInterval(()=> {
      if(wrapperDom.clientHeight > 0) {
        clearInterval(checker);
        this.ol = new Map({
        target: this.$refs.olWrapper,
        layers: [
          this.layers.iiif,
          this.layers.geoMap,
          this.layers.staticImage
        ],
        view: new View({
          center: [0, 0],
          zoom: 0
        })
      });
    this.validateScreenplay()
      .then(() => { this.changeScene(-1, true); })
      .catch(() => { this.inError = true; });
      
      } else {
        console.log("waiting");
      }
    }, 100);
    
    

    
  }
}
</script>

<style>
@import '~@fortawesome/fontawesome-svg-core/styles.css';
@import '~ol/ol.css';
@import url('https://fonts.googleapis.com/css2?family=Epilogue:ital,wght@0,400;0,700;1,400&display=swap');


#error {
  text-align: center;
  padding-top: 1.0em;
}

#app {
  width: 100%;
  height: 100%;
  background-color: #dcdcdc;
  position: relative;
  font-family: 'Epilogue', sans-serif;
  font-size: 1.0rem;
}

figure {
  margin: 0
}

.hidden {
  display: none;
}

#ol-wrapper {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 1;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif, "Apple Color Emoji", "Segoe UI Emoji", "Segoe UI Symbol";
}

#caption-wrapper {
  position: absolute;
  bottom: 10px;
  left: 10px;
  right: 10px;
  z-index: 2;
  display: flex;
  flex-direction: row;
  background-color: rgba(253, 249, 243, 0.96);
  border-radius: 3px;
}

#caption-shower {
  position: absolute;
  bottom: 0;
  left: 40px;
  width: auto;
  padding: 10px 10px 5px;
  background-color: rgba(253, 249, 243, 0.96);
  font-size: 0.85rem;
  color: rgb(163, 158, 148);
  z-index: 2;
}

.begin-callout { 
  font-weight: bold;
  font-size: 1.2rem;
  text-align: right;
  cursor: pointer; }

.begin-callout:hover {
  color: rgb(104, 18, 18);
}

.caption-control {
  min-width: 2.0em;
  display: flex;
  flex-direction: column;
  justify-content: flex-end;
}


.caption-control > div {
  display: inline-block;
  cursor: pointer;
  font-size: 1.2rem;
  transition: color 0.4s;
}

.caption-control > div:hover {
  color: rgb(104, 18, 18);
}

#caption-left-control {
  border-right: 2px dotted rgba(238, 232, 221, 0.99);
}

#caption-right-control {
  border-left: 2px dotted rgba(238, 232, 221, 0.99);
}

#caption-left-control > div {
  position: absolute;
  left: 5px;
  bottom: 10px;
  text-align: left;
}

#caption-right-control > div {
  position: absolute;
  right: 5px;
  bottom: 10px;
  text-align: right;
}

#caption-center {
  flex-grow: 1;
  padding: 20px 20px 10px;
}

#caption-center a {
  color: black;
  text-decoration: none;
  background-image: linear-gradient(0deg, rgba(163, 158, 148, 0.7) 39%, rgba(0, 0, 0, 0) 0px);
}

#caption-center a:hover {
  background-image: linear-gradient(0deg, rgba(104, 18, 18, 0.3) 39%, rgba(0, 0, 0, 0) 0px);
}

#caption-bottom-control {
  font-size: 0.85rem;
  color: rgb(163, 158, 148);
}

.control-bottom-icon {
  transition: color 0.4s;
}

.control-bottom-icon:not(:last-of-type){
    margin-right: 7px;
}

.control-bottom-icon:hover {
  cursor: pointer;
  color: rgb(104, 18, 18);
}

h1.title {
  margin: 0 0 10px 0;
  font-weight: 700;
  text-shadow: 2px 2px 0px rgba(131, 126, 118, 0.3)

}

h2.subtitle {
  margin: 0 0 8px 0;
  font-size: 1.3rem;
  font-weight: 400;
}

h3.author {
  margin: 0;
  font-size: 1.0rem;
  font-weight: 400;
}

.caption-title {
  margin: 0 0 5px 0;
  font-size: 1.2rem;
}

#caption-center p {
  margin: 0 0 5px 0;
}

.counter {
  margin: 0 10px 0 0;
  font-weight: 700;
  font-size: 0.8rem;
}

#caption-center p.small-credit {
  font-size: 0.9rem;
  margin: 15px 0 0 0;
  color: rgb(163, 158, 148);
}

.ol-attribution.ol-uncollapsible {
  top: 0;
  right: 0;
  bottom: unset;
  border-radius: 0;
  font-size: 0.7rem;
}

.ol-wrapper-faded {
  opacity: 0.2;
}
</style>
