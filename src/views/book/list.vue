<template>
  <div class="app-container">
    <div class="filter-container">
      <el-input
        v-model="listQuery.title"
        placeholder="书名"
        style="width: 200px"
        class="filter-item"
        clearable
        @keyup.enter.native="handleFilter"
        @clear="handleFilter"
        @blur="handleFilter"
      />
      <el-input
        v-model="listQuery.author"
        placeholder="作者"
        style="width: 200px"
        class="filter-item"
        clearable
        @keyup.enter.native="handleFilter"
        @clear="handleFilter"
        @blur="handleFilter"
      />
      <el-select
        v-model="listQuery.category"
        placeholder="分类"
        clearable
        class="filter-item"
        @change="handleFilter"
      >
        <el-option v-for="category in categoryList" :key="category.value" :value="category.value" :label="category.label">{{ category.label }}({{ category.num }})</el-option>
      </el-select>
      <el-button v-waves class="filter-item" type="primary" icon="el-icon-search" style="margin-left: 10px" @click="handleFilter">查询</el-button>
      <el-button class="filter-item" type="primary" icon="el-icon-edit" style="margin-left: 5px" @click="handleCreate">创建</el-button>
      <el-checkbox v-model="showCover" class="filter-item" style="margin-left: 5px" @change="changeShowCover">显示封面</el-checkbox>
    </div>
    <el-table
      v-loading="listLoading"
      :data="list"
      border
      fit
      highlight-current-row
      style="width: 100%"
      :default-sort="defaultSort"
      @sort-change="sortChange"
    >
      <el-table-column prop="id" sortable="custom" align="center" width="80" label="ID" />
      <el-table-column prop="title" sortable="custom" align="center" width="150" label="书名">
        <template slot-scope="{ row: { titleWrapper }}">
          <span v-html="titleWrapper" />
        </template>
      </el-table-column>
      <el-table-column prop="author" sortable="custom" align="center" width="150" label="作者">
        <template slot-scope="{ row: { authorWrapper }}">
          <span v-html="authorWrapper" />
        </template>
      </el-table-column>
      <el-table-column prop="publisher" sortable="custom" align="center" width="150" label="出版社" />
      <el-table-column prop="categoryText" sortable="custom" align="center" width="100" label="分类" />
      <el-table-column prop="language" sortable="custom" align="center" label="语言" />
      <el-table-column v-if="showCover" prop="cover" sortable="custom" align="center" width="150" label="封面">
        <template slot-scope="scope">
          <a :href="scope.row.cover" target="_blank">
            <img :src="scope.row.cover" style="width: 120px;height: 180px">
          </a>
        </template>
      </el-table-column>
      <el-table-column prop="fileName" sortable="custom" align="center" width="100" label="文件名" :formatter="emptyColumn" />
      <el-table-column prop="filePath" sortable="custom" align="center" width="100" label="文件路径" :formatter="emptyColumn" />
      <el-table-column prop="coverPath" sortable="custom" align="center" width="100" label="封面路径" :formatter="emptyColumn" />
      <el-table-column prop="unzipPath" sortable="custom" align="center" width="100" label="解压路径" :formatter="emptyColumn" />
      <el-table-column prop="createUser" sortable="custom" align="center" width="100" label="上传人" :formatter="emptyColumn" />
      <el-table-column prop="createDt" sortable="custom" align="center" width="100" label="上传时间" :formatter="timeFilter" />
      <el-table-column sortable="custom" align="center" width="120" fixed="right" label="操作">
        <template slot-scope="{ row }">
          <el-button type="text" icon="el-icon-edit" @click="handleUpdate(row)" />
          <el-button type="text" icon="el-icon-delete" style="color: #f56c6c" @click="handleDelete(row)" />
        </template>
      </el-table-column>
    </el-table>
    <pagination v-show="listCount > 0" :total="listCount" :page.sync="listQuery.page" :limit.sync="listQuery.pageSize" @pagination="refresh" />
  </div>
</template>

<script>
import { deleteBook, getCategory, listBook } from '@/api/book'
import Pagination from '../../components/Pagination/index'
import waves from '../../directive/waves/waves'
import { parseTime } from '@/utils'

export default {
  components: {
    Pagination
  },
  directives: { waves },
  data() {
    return {
      listCount: 0,
      listQuery: {
        page: 1,
        pageSize: 20
      },
      showCover: false,
      categoryList: [],
      listLoading: false,
      list: [],
      defaultSort: []
    }
  },
  created() {
    this.parseQuery()
  },
  mounted() {
    this.getList()
    this.getCategoryList()
  },
  beforeRouteUpdate(to, from, next) {
    if (to.path === from.path) {
      this.getList()
    }
    next()
  },
  methods: {
    emptyColumn(row, col, val) {
      return val || '无'
    },
    timeFilter(row, col, time) {
      return time ? parseTime(time, '{y}-{m}-{d} {h}:{i}') : '无'
    },
    parseQuery() {
      const query = { ...this.$route.query }
      let sort = '+id'
      const listQuery = {
        page: 1,
        pageSize: 20,
        sort: '+id'
      }
      query.page && (query.page = +query.page)
      query.pageSize && (query.pageSize = +query.pageSize)
      query.sort && (sort = query.sort)
      const sortSymbol = sort[0]
      const sortColumn = sort.slice(1, sort.length)
      this.defaultSort = {
        prop: sortColumn,
        order: sortSymbol === '+' ? 'ascending' : 'descending'
      }
      this.listQuery = { ...listQuery, ...query }
    },
    wrapperKeyword(k, v) {
      function highlight(value) {
        return `<span style="color: #1890ff">${value}</span>`
      }
      if (!this.listQuery[k]) {
        return v
      } else {
        return v.replace(new RegExp(this.listQuery[k], 'ig'), v => highlight(v))
      }
    },
    getList() {
      this.listLoading = true
      listBook(this.listQuery).then(res => {
        console.log(res)
        const { list, count } = res.data
        this.list = list
        this.list.forEach(book => {
          book.titleWrapper = this.wrapperKeyword('title', book.title)
          book.authorWrapper = this.wrapperKeyword('author', book.author)
        })
        this.listCount = count
      }).finally(() => {
        this.listLoading = false
      })
    },
    getCategoryList() {
      getCategory().then(res => {
        this.categoryList = res.data
      })
    },
    refresh() {
      this.$router.push({
        path: '/book/list',
        query: this.listQuery
      })
    },
    handleFilter() {
      console.log('query')
      this.listQuery.page = 1
      this.refresh()
    },
    handleCreate() {
      this.$router.push('/book/create')
    },
    handleUpdate(row) {
      this.$router.push(`/book/edit/${row.fileName}`)
    },
    handleDelete(row) {
      this.$confirm('此操作将永久删除该电子书，是否继续？', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        deleteBook(row.fileName).then(res => {
          console.log(res)
          this.$notify({
            title: '成功',
            message: res.msg || '删除成功',
            type: 'success',
            duration: 2000
          })
          this.handleFilter()
        })
      })
    },
    changeShowCover(val) {
      this.showCover = val
    },
    sortChange(data) {
      console.log(data)
      const { prop, order } = data
      this.sortBy(prop, order)
    },
    sortBy(prop, order) {
      if (order === 'ascending') {
        this.listQuery.sort = `+${prop}`
      } else {
        this.listQuery.sort = `-${prop}`
      }
      this.handleFilter()
    }
  }
}
</script>
