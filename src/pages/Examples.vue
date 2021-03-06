<template>
  <div class="wrapper">
    <list class="example-list" v-if="examples && examples.length">
      <template v-for="(exampleGroup, n) in currentTab.group">
        <cell class="group-info"
          v-if="exampleGroup && exampleGroup.title || exampleGroup.name"
          :ref="exampleGroup.type"
          :key="exampleGroup.type">
          <text class="group-title">{{i18n(exampleGroup.title || exampleGroup.name)}}</text>
          <text class="group-desc" v-if="exampleGroup.desc">{{i18n(exampleGroup.desc)}}</text>
          <text class="doc-link"
            v-if="exampleGroup.desc && exampleGroup.docLink"
            @click="jumpTo(exampleGroup.docLink)"
            >{{i18n(tips.SEE_MORE)}} >></text>
        </cell>
        <cell class="section"
          v-for="(row, i) in divideArary(exampleGroup.examples)"
          :key="exampleGroup.type + '-row-' + i">
          <div class="example-row">
            <div class="example-box"
              v-for="example in row"
              :key="'title' + i18n(example.title)">
              <text class="example-title">{{i18n(example.title)}}</text>
            </div>
          </div>
          <div class="example-row">
            <div class="example-box"
              v-for="example in row"
              :key="i18n(example.title)">
              <div style="align-items: center">
                <a :href="i18n(example.hash) | url">
                  <image class="screenshot" :src="i18n(example.screenshot)" />
                </a>
                <text class="example-tips"
                  @click="viewSource(example.hash)"
                  >{{i18n(tips.VIEW_SOURCE)}}</text>
              </div>
            </div>
          </div>
        </cell>
        <cell class="section-gap"
          v-if="n < currentTab.group.length - 1"
          :key="exampleGroup.type + '-gap'"></cell>
      </template>
    </list>
    <div class="loading" v-else-if="showLoading">
      <text class="loading-text">loading ...</text>
    </div>
    <div class="tabbar" v-if="tabs && tabs.length">
      <div v-for="tab in tabs" :key="tab.type"
        :class="['tab-cell', tab.type === activeTab ? 'active-tab-cell': '']"
        @click="toggleTab(tab.type)">
        <text :class="[
          'tab-name',
          `tab-name-${language}`,
          tab.type === activeTab ? `active-tab-name-${language}`: ''
        ]">{{i18n(tab.name)}}</text>
      </div>
    </div>
  </div>
</template>

<style scoped>
  .example-list {
    width: 750px;
    position: absolute;
    top: 0;
    bottom: 100px;
  }
  .loading {
    flex: 1;
    justify-content: center;
    align-items: center;
  }
  .loading-text {
    font-size: 60px;
    color: #BBB;
  }
  .group-title {
    width: 750px;
    padding-top: 20px;
    padding-bottom: 20px;
    font-size: 40px;
    text-align: center;
    color: #00B4FF;
    background-color: #EAF8FD;
  }
  .group-desc {
    font-size: 28px;
    color: #999;
    margin-top: 20px;
    margin-left: 30px;
    margin-right: 40px;
  }
  .doc-link {
    font-size: 26px;
    color: rgba(0, 189, 255, 0.6);
    text-align: right;
    margin-top: 10px;
    margin-right: 60px;
    margin-bottom: 20px;
  }
  .section {
    padding-top: 20px;
    padding-bottom: 20px;
  }
  .section-gap {
    height: 70px;
  }
  .example-row {
    flex-direction: row;
    align-items: center;
  }
  .example-box {
    justify-content: space-between;
    align-items: center;
    /* padding-left: 15px;
    padding-right: 15px;
    width: 374px; */
    padding-left: 6px;
    padding-right: 6px;
    width: 250px;
  }
  .screenshot {
    /* width: 290px;
    height: 453px; */
    width: 220px;
    height: 343px;
    border-width: 1px;
    border-color: #DDD;
  }
  .example-title {
    font-size: 32px;
    text-align: center;
    color: #606060;
    padding-top: 10px;
    padding-bottom: 10px;
  }
  .example-tips {
    font-size: 26px;
    text-align: center;
    color: #999;
    padding-top: 10px;
    padding-bottom: 10px;
  }
  .tabbar {
    width: 750px;
    position: fixed;
    bottom: 0;
    height: 100px;
    flex-direction: row;
    justify-content: space-around;
    align-items: flex-end;
    background-color: #E6E6E6;
  }
  .tab-cell {
    width: 186px;
    height: 100px;
    border-top-width: 2px;
    border-top-style: solid;
    border-top-color: #DDDDDD;
    justify-content: center;
    background-color: #FBFBFB;
  }
  .active-tab-cell {
    border-top-color: rgba(0, 189, 255, 0.8);
    background-color: #BDECFF;
  }
  .tab-name {
    text-align: center;
    color: #666666;
  }
  .tab-name-zh {
    font-size: 36px;
  }
  .tab-name-en {
    font-size: 32px;
  }
  .active-tab-name-zh {
    color: #00B4FF;
    font-size: 45px;
    font-weight: bold;
  }
  .active-tab-name-en {
    color: #00B4FF;
    font-size: 34px;
    font-weight: bold;
  }
</style>

<script>
  import { fetchExamples, getLanguage } from '../shared/utils'
  // import getExamples from '../../examples'
  // const exampleMap = getExamples({ scope: 'mobile', filterTODO: true })
  const exampleMap = []

  const navigator = weex.requireModule('navigator')
  const storage = weex.requireModule('storage')
  const picker = weex.requireModule('picker')
  const examplesKey = 'WEEX_PLAYGROUND_APP_EXAMPLES'
  let useStorage = false
  export default {
    data () {
      return {
        examples: exampleMap,
        showLoading: false,
        language: 'en',
        activeTab: 'component',
        activeGroup: 'div',
        tips: {
          VIEW_SOURCE: { en: 'view source', zh: '查看源码' },
          LANGUAGE: { en: 'Language', zh: '语言' },
          SEE_MORE: { en: 'see more', zh: '查看更多' }
        }
      }
    },
    beforeCreate () {
      getLanguage(language => {
        this.language = language
      })

      // read examples from storage
      storage.getItem(examplesKey, event => {
        if (event.result === 'success') {
          const data = JSON.parse(event.data)
          if (data && Array.isArray(data.examples)) {s
            this.examples = data.examples
            if (WXEnvironment.platform.toLowwerCase() !== 'web') {
              useStorage = true
            }
          }
        }
      })

      // update examples to storage
      fetchExamples(result => {
        result.timestamp = Date.now()
        storage.setItem(examplesKey, JSON.stringify(result))
        if (!useStorage) {
          this.examples = result.examples
        }
      })

      setTimeout(() => { this.showLoading = true }, 400);
    },
    computed: {
      tabs () {
        return this.examples.map(group => ({
          type: group.type,
          name: group.name
        }))
      },
      currentTab () {
        return this.examples.filter(tab => tab.type === this.activeTab)[0]
      }
    },
    methods: {
      divideArary (array) {
        const column = 3
        const result = []
        for (let i = 0; i < array.length; ++i) {
          const idx = Math.floor(i/column)
          if (!result[idx]) {
            result[idx] = []
          }
          result[idx].push(array[i])
        }
        return result
      },
      toggleTab (tabType) {
        this.activeTab = tabType
        this.activeGroup = this.currentTab.group[0].type
      }
    },
    beforeDestroy () {
      storage.removeItem('CURRENT_DOCUMENT_URL')
      storage.removeItem('CURRENT_SOURCE_HASH')
    }
  }
</script>
