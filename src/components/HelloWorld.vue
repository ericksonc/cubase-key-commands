<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
    <div>
      <input
        type="text"
        id="search"
        placeholder="Type here..."
        @input="parseInput"
        v-model="searchTerm"
      >
    </div>
    <br><br>
    <div v-for="category in filteredCategoryList" :key="category.category">
      <div class="category">{{category.category}}</div>
      <div v-for="commandItem in category.items" :key="commandItem.name" class="command-item">
        <div>{{commandItem.name}}</div>
        <div>{{commandItem.command}}</div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import {onMounted, ref, computed} from 'vue'
const msg = "foo"
import axios from 'axios'
const searchTerm = ref('')
let xmlData = ref("Loading XML...")
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

function parseItem(el: element): oItem | false {
  const elements = getElements(el)
  if (!elements) return false
  const name = elements[0]["attributes"]["value"]
  let command: string | null = null
  if (elements.length > 1) {
    command = elements[1]["attributes"]["value"]
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

function parseInput(input: string) {
  console.log(searchTerm)
}

onMounted(() => {
    axios.get('http://localhost:8080/Key Commands.xml')
                  .then(response => {
                      xmlData.value = response.data;
                      console.log(xmlData)
                      jsonData.value = convert.xml2json(xmlData.value, {compact: false, spaces: 4});
                      objectData.value = JSON.parse(jsonData.value)
                      parseCommands(objectData.value as unknown as KeyCommandsFile)

    });
  }
)

</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
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
