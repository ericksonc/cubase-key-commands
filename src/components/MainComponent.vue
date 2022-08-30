<template>
  <div class="hello">
    <h1>Cubase Key Commands</h1>
    <div v-if="xmlData == ''">
      Welcome to Cubase Key Commands Filter / Seaerch!

      Start by locating your "Key Commands.xml" file:
      <p></p>
      <input type="file" accept=".xml" id="inputfile" @change="processFile" />

      <p><em>Why do I need to upload this file?</em></p>
      <p>This app needs to see the contents of your Key Commands.xml to show you which key commands you currently have set.</p>
    </div>
    <div id = "commands-table" v-else>
      <div>
        <input
          type="text"
          id="search"
          placeholder="Type here..."
          @input="parseInput"
          v-model="searchTerm"
        >
      </div>
      <div>
        <input 
          type="checkbox"
          v-model="fullPhrase"
        > Match full phrase
      </div>
      <br><br>
      <div v-for="category in filteredCategoryList" :key="category.category" class="category-list">
        <div class="category">{{category.category}}</div>
        <div v-for="commandItem in category.items" :key="commandItem.name" class="command-item">
          <div class="command-name">{{commandItem.name}}</div>
          <div class="command-key">{{commandItem.command}}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import {onMounted, ref, computed} from 'vue'
const msg = "foo"
import axios from 'axios'
const searchTerm = ref('')
const fullPhrase = ref(false)
let xmlData = ref("")
let jsonData = ref("")
let objectData = ref({})

import convert from "xml-js"

// type elementArray

type attributes = {
  name: string
  value: string
}

type element = {
  type: "element";
  name: string
  attributes: attributes
  elements?: element[]
}

interface KeyCommandsFile {
  elements: element[]
}

interface oItem {
  name: string
  command: string | null
}

interface oCategory {
  category: string
  items: oItem[]
}


function isElement(obj: element | object | false): obj is element {
  if (!obj) return false
   return "type" in obj && obj["type"] == "element";
}

function getElements(el: Array<element> | object | false): element[] | false {
  if (!isElement(el)) return false
  if ("elements" in el && el["elements"] !== undefined && el["elements"].length > 0) return el["elements"]
  else return false
}

const defaultFilteredValue: oCategory[] = [
  {category: "foo",
  items: [
    {
      name: "bar",
      command: "escape"
    }
  ]}
]

const categoryList: oCategory[] = []
const filteredCategoryList = ref(defaultFilteredValue)

function processFile(event: Event) {
  if (!event || !event.target || !event.target) return false
  if (event == null) return false
  const fileInput = (event.target as HTMLInputElement)
  const xmlFile = fileInput.files && fileInput.files.length ? fileInput.files[0] : null
  if (!xmlFile) {
    console.error("Couldn't resolve file")
    return false
  }
  console.log(xmlFile)

  const reader = new FileReader();   

  reader.onload = () => {
    if (typeof reader.result !== "string") {
      console.error("It wasn't a string")
      return false
    }
    xmlData.value = reader.result;
    parseXml()
  }

  const xmlText = reader.readAsText(xmlFile)

  console.log(xmlText)
}

function parseXml() {
  jsonData.value = convert.xml2json(xmlData.value, {compact: false, spaces: 4});
  objectData.value = JSON.parse(jsonData.value)
  parseCommands(objectData.value as unknown as KeyCommandsFile)
}

function parseItem(el: element): oItem | false {
  const elements = getElements(el)
  if (!elements) return false
  const name = elements[0]["attributes"]["value"]
  let command = ""
  if (elements.length > 1) {
    const keys = elements[1]
    if (keys["name"] == "string") command = keys["attributes"]["value"]
    else if(keys["name"] == "list") {
      const keyArr = keys["elements"] as unknown as element[]
  
      (keyArr).forEach(el => { 
        if (command !== "") command = command.concat(", ")
        command = command.concat(el["attributes"]["value"])
      })
    }
    else console.error(`Couldn't detect keys for command ${name}`)
  }
  return {name, command}
}

function parseCommands(obj: KeyCommandsFile) {
  const keyCommandsElements = obj["elements"][0]
  if (isElement(keyCommandsElements)) {
    const top = getElements(keyCommandsElements)
    if (!top) return false
    const categories = getElements(top[0])
    if (!categories) return false

    for (let i = 0; i < categories.length; i++) {
      const items: oItem[] = [] 
      const category = categories[i]
      const categoryElements = getElements(category)
      if (!categoryElements) return false
      const categoryTitle = categoryElements[0]["attributes"]["value"]
      categoryList.push()
      const commandList = getElements(categoryElements[1])
      if (!commandList) return false
      for (let i = 0; i < commandList.length; i++) {
        const command = commandList[i]
        const oItem = parseItem(command)
        if (oItem) items.push(oItem)
      }
      categoryList.push({category: categoryTitle, items})
      filteredCategoryList.value = categoryList

    }
  }

}

function parseInput() {
  console.log(searchTerm)
  if (searchTerm.value == "") {
    filteredCategoryList.value = categoryList
    return
  }
  const newFilteredList: oCategory[] = []
  for (let i = 0; i < categoryList.length; i++) {
    const items: oItem[] = []
    const category = categoryList[i]
    for (let j = 0; j < category.items.length; j++) {
      const item = category.items[j]
      if (checkMatch(item.name)) items.push(item)
    }
    if (items.length > 0) newFilteredList.push({
      category: category.category,
      items
    })
  }
  
  filteredCategoryList.value = newFilteredList
}

function checkMatch(label:string): boolean {
  if (fullPhrase.value) {
     return label.toLowerCase().includes(searchTerm.value.toLowerCase())
  } else {
    const words: string[] = searchTerm.value.split(" ")
    if (words.length == 0) return false
    let match = true
    for (let i = 0; i < words.length; i++) {
      if (!label.toLowerCase().includes(words[i].toLowerCase())) match = false
    }
    return match
  }
}

</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>

#search {
   width: 200px; 
   height: 20px;
   font-size: 18px;
}

.category-list {
  width:600px;
}

.category {
  font-size: 150%;
  font-weight: bold;
}

.command-item {
  display: grid;
  grid-template-columns: 2fr 1fr;
  grid-gap: 20px;
}

.command-name {  
  padding: 3px 3px 3px 3px;
  border:1px dashed cadetblue;
}
.command-key {
  padding: 3px 3px 3px 3px;
  border:1px dashed cadetblue;
  color: gray;
}

h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
