<template>
  <div class="d-flex flex-column align-items-center vh-100">
    <b-button @click="downloadExcel" variant="success" class="align-self-center"
      >Download Excel</b-button
    >
    <b-table striped hover :items="flattenedItems" :fields="fields" class="w-75 mb-3">
      <template #cell(action)="data">
        <b-button @click="openModal(data.item.id)" variant="primary">Add</b-button>
        <b-button @click="deleteItem(data.item.id)" variant="danger">Delete</b-button>
      </template>
    </b-table>
    <b-button @click="openModal(null)" variant="primary" class="align-self-center">Add</b-button>
    <b-modal v-model="showModal" title="Add Autopart" @ok="addItem">
      <b-form @submit.prevent="addItem">
        <b-form-group label="Detail Name">
          <b-form-input v-model="newItem.detailName" required></b-form-input>
        </b-form-group>
        <b-form-group label="Price">
          <b-form-input type="number" v-model="newItem.price" required></b-form-input>
        </b-form-group>
        <b-form-group label="Quantity">
          <b-form-input type="number" v-model="newItem.quantity" required></b-form-input>
        </b-form-group>
      </b-form>
    </b-modal>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import axios from 'axios'
import { BTable, BButton, BModal, BForm, BFormGroup, BFormInput } from 'bootstrap-vue'

class Autopart {
  constructor(
    public id: number,
    public detailName: string,
    public price: number,
    public quantity: number,
    public total: number,
    public children: Autopart[]
  ) {}
}

const fields = ['#', 'detailName', 'price', 'quantity', 'total', 'action']
const items = ref<Autopart[]>([])
const showModal = ref(false)
const newItem = ref({
  parentId: null,
  detailName: '',
  price: 0,
  quantity: 0
})

onMounted(async () => {
  const response = await axios.get('http://localhost:8090/api/v1/autopart/list')
  items.value = response.data.map(
    (item: any) =>
      new Autopart(item.id, item.detailName, item.price, item.quantity, item.total, item.children)
  )
})

const deleteItem = async (id: number) => {
  await axios.delete(`http://localhost:8090/api/v1/autopart/${id}`)

  function removeChildItems(item) {
    if (item.children) {
      item.children = item.children.filter((child) => {
        if (child.id === id) return false
        removeChildItems(child)
        return true
      })
    }
    return item.id !== id
  }

  items.value = items.value.filter(removeChildItems)
}

const addItem = async () => {
  await axios.post('http://localhost:8090/api/v1/autopart', newItem.value)

  newItem.value = {
    parentId: null,
    detailName: '',
    price: 0,
    quantity: 0
  }

  const response = await axios.get('http://localhost:8090/api/v1/autopart/list')
  items.value = response.data.map(
    (item: any) =>
      new Autopart(item.id, item.detailName, item.price, item.quantity, item.total, item.children)
  )

  showModal.value = false
}

const openModal = (parentId: number | null) => {
  newItem.value.parentId = parentId
  showModal.value = true
}

const downloadExcel = async () => {
  const response = await axios.get('http://localhost:8090/api/v1/autopart/excel', {
    responseType: 'blob'
  })
  const url = window.URL.createObjectURL(new Blob([response.data]))
  const link = document.createElement('a')
  link.href = url
  link.setAttribute('download', 'autopart.xlsx')
  document.body.appendChild(link)
  link.click()
}

const flattenItems = (items: Autopart[], prefix = ''): any[] => {
  let count = 1
  let result: any[] = []
  for (const item of items) {
    result.push({
      '#': prefix + count,
      ...item
    })
    result = result.concat(flattenItems(item.children, prefix + count + '.'))
    count++
  }

  return result
}

const flattenedItems = computed(() => flattenItems(items.value))
</script>
