<template>
  <div>
    <Layout class="main-layout">
      <Sider
        ref="mainSider"
        class="sider-bar"
        hide-trigger
        collapsible
        :collapsed-width="50"
        v-model="isCollapsed"
      >
        <div class="logo">
          <img src="@/assets/lyrebird.logo.png" />
          <span>{{logo}}</span>
        </div>
        <Divider class="sider-bar-divider" />
        <Menu theme="dark" width="auto" :class="menuitemClasses" :active-name="activeName" ref="menu">
          <div v-for="(menuItem, index) in menu" :key="index">
            <Tooltip
              :content="menuItem.title"
              placement="right"
              :disabled="!isCollapsed"
              transfer
              style="width: 100%"
            >
              <MenuItem
                :name="menuItem.title"
                @click.native="menuItemOnClick(menuItem)"
              >
                <b>{{menuItemTitle(menuItem)}}</b>
              </MenuItem>
            </Tooltip>
          </div>
        </Menu>
      </Sider>
      <Layout>
        <Header class="main-header" inline>
          <Icon type="md-menu" color="white" size="24" @click.native="collapsedSider"></Icon>
          <notice-center></notice-center>
        </Header>
        <Content>
          <div class="main-container">
            <router-view></router-view>
          </div>
        </Content>
        <Footer class="main-footer">
          <span class="main-footer-status-placeholder"></span>
          <span v-show="activatedGroupName" class="main-footer-status-no-pointer">
            <b>Activated mock group: {{activatedGroupName}}</b>
            <Icon type="md-close-circle" style="cursor:pointer;" @click="resetActivatedData" />
          </span>
          <StatusBar />
          <span class="main-footer-right">
            <Poptip
              content="content"
              placement="top-start"
              class="main-footer-status"
              width="250"
            >
              <b class="main-footer-status-button">Bandwidth: {{bandwidthExplanation}} </b>
              <div slot="title">
                <b>Bandwidth</b>
              </div>
              <div slot="content">
                <Row type="flex" justify="space-around">
                  <Col span="12" v-for="(item, index) in bandwidthTemplates" :key="index">
                    <Button
                      style="min-width:95px;margin-top:5px;"
                      :class="item.bandwidth == bandwidth ? 'bandwidth-btn-highlight' : ''"
                      @click.prevent="updateBandwidth(item.template_name)"
                    >{{ item.template_name }}</Button>
                  </Col>
                </Row>
              </div>
            </Poptip>
            <Poptip
              v-if="status"
              content="content"
              placement="top-end"
              class="main-footer-status"
              width="250"
            >
              <span class="main-footer-status-button">
                <Icon type="ios-arrow-up" style="padding-right:3px;"/>
                <b>Version {{status.version}}</b>
              </span>
              <div slot="title">
                <b>Lyrebird {{status.version}}</b>
              </div>
              <div slot="content">
                <Row v-for="key in showedStatus" :key="key">
                  <i-col span="11">
                    <b style="float: right">{{key.toUpperCase()}}</b>
                  </i-col>
                  <i-col span="12" offset="1">{{status[key]}}</i-col>
                </Row>
                <Divider style="margin:10px 0;"/>
                <div style="text-align:center">
                  <strong>
                    Copyright &copy; 2018-present 
                    <a href="https://meituan-dianping.github.io/lyrebird" target="_blank" >Meituan</a>.
                  </strong>
                </div>
              </div>
            </Poptip>
            <span class="main-footer-status">
              <a
                href="https://github.com/Meituan-Dianping/lyrebird/issues/new?assignees=&labels=&template=bug_report.md&title="
                target="_blank"
              >
                <Icon type="ios-bug" class="main-footer-status-button"/>
              </a>
            </span>
            <span class="main-footer-status-placeholder"></span>
          </span>
        </Footer>
      </Layout>
    </Layout>
  </div>
</template>

<script>
import NoticeCenter from '@/views/notice/NoticeCenter.vue'
import StatusBar from '@/views/statusbar/StatusBar.vue'

export default {
  name: 'MainLayout',
  components: {
    NoticeCenter,
    StatusBar
  },
  data () {
    return {
      isCollapsed: true,
      showedStatus: ["ip", "mock.port", "proxy.port"]
    }
  },
  mounted () {
    this.$store.dispatch('loadMenu')
    this.$store.dispatch('loadStatus')
    this.$store.dispatch('loadManifest')
    this.$store.dispatch('loadActivatedGroup')
    this.$store.dispatch('loadBandwidth')
    this.$store.dispatch('loadBandwidthTemplates')
    this._keydownListener = (e) => {
      this.$bus.$emit('keydown', e)
    }
    document.addEventListener('keydown', this._keydownListener)
  },
  beforeDestroy () {
    document.removeEventListener('keydown', this._keydownListener)
  },
  created () {
    this.$bus.$on('msg.success', this.successMessage)
    this.$bus.$on('msg.info', this.infoMessage)
    this.$bus.$on('msg.error', this.errorMessage)
  },
  watch: {
    activeMenuItem: function (newValue, oldValue) {
      this.refreshPage(newValue)
      // :active-name 异步刷新后，需要手动更新 
      // https://github.com/iview/iview/issues/1245#issuecomment-352992001 
      this.$nextTick(function () {
        this.$refs.menu.updateActiveName()
      })
    }
  },
  computed: {
    menuitemClasses () {
      return ["menu-item", this.isCollapsed ? "collapsed-menu" : "menu"]
    },
    logo () {
      if (this.isCollapsed) {
        return ''
      } else {
        return 'Lyrebird'
      }
    },
    menu () {
      return this.$store.state.menu
    },
    status () {
      return this.$store.state.status
    },
    manifest () {
      return this.$store.state.manifest
    },
    activeName () {
      return this.$store.state.activeName
    },
    bandwidth () {
      return this.$store.state.bandwidth.bandwidth
    },
    bandwidthTemplates () {
      return this.$store.state.bandwidth.bandwidthTemplates
    },
    bandwidthExplanation () {
      for (let v of this.bandwidthTemplates) {
        if (this.bandwidth == v['bandwidth']) {
          if (this.bandwidth == -1) {
            return v['template_name']
          }
          else {
            return `${v['template_name']} ( ${v['bandwidth']} Kb/s)`
          }
        }
      }
    },
    activeMenuItem () {
      return this.$store.state.activeMenuItem
    },
    activatedGroupName () {
      const activatedGroups = this.$store.state.inspector.activatedGroup
      if (activatedGroups === null) {
        return null
      }
      if (Object.keys(activatedGroups) === 0) {
        return null
      }
      let text = ''
      for (const groupId in activatedGroups) {
        text = text + activatedGroups[groupId].name + ' '
      }
      return text
    }
  },
  methods: {
    collapsedSider () {
      this.$refs.mainSider.toggleCollapse()
    },
    menuItemTitle (menuItem) {
      if (this.isCollapsed) {
        return menuItem.title.substring(0, 1)
      } else {
        return menuItem.title
      }
    },
    menuItemOnClick (menuItem) {
      // 更新activeName 与 activeMenuItem
      // 点击后，activeMenuItem更新，触发watch，操作页面更新
      this.$store.dispatch('updateActiveMenuItem', menuItem)
    },
    refreshPage (menuItem) {
      // 更新 router
      if (menuItem.type === 'router') {
        if (menuItem.name === 'plugin-view' || menuItem.name === 'plugin-container') {
          this.$store.commit('plugin/setSrc', menuItem.params.src)
        }
        this.$router.push({ name: menuItem.name, params: menuItem.params })
      } else {
        window.open(menuItem.path, '_self')
      }
    },
    resetActivatedData () {
      this.$store.dispatch('deactivateGroup')
    },
    successMessage (msg) {
      this.$Message.success({
        content: msg,
        duration: 3,
        closable: true
      })
    },
    infoMessage (msg) {
      this.$Message.info({
        content: msg,
        duration: 3,
        closable: true
      })
    },
    errorMessage (msg) {
      this.$Message.error({
        content: msg,
        duration: 0,
        closable: true
      })
    },
    updateBandwidth (template_name) {
      this.$store.dispatch('updateBandwidth', template_name)
    }
  }
}
</script>

<style scoped>
.main-layout {
  height: 100vh;
}
.sider-bar {
  background-color: #515a6e;
}
.sider-bar-divider {
  height: 1px;
  margin: 0;
  background: #6c6c6c;
}
.logo {
  height: 38px;
  display: flex;
  align-items: center;
  justify-content: center;
}
.logo span {
  color: white;
  font-size: 25px;
  font-weight: bolder;
  font-style: italic;
}
.logo img {
  width: 32px;
}
.main-header {
  height: 38px;
  line-height: 38px;
  padding: 0;
  margin: 0;
  background-color: #0fccbf;
}
.main-footer {
  height: 28px;
  line-height: 28px;
  padding: 0;
  background-color: #0fccbf;
}
.main-footer-status-placeholder {
  margin-left: 5px;
}
.main-footer-right {
  float: right;
}
.collapsed-menu span {
  width: 0px;
  transition: width 0.2s ease;
}
.menu-item span {
  display: inline-block;
  overflow: hidden;
  width: 69px;
  text-overflow: ellipsis;
  white-space: nowrap;
  vertical-align: bottom;
  transition: width 0.2s ease 0.2s;
}
.main-container {
  height: calc(100vh - 66px);
  background: #fff;
}
.bandwidth-btn-highlight {
  background-color: #0fccbf !important;
  color: #fff;
  outline: none;
}
</style>

<style>
.ivu-split-pane {
  overflow: hidden;
}
.main-footer-status-no-pointer {
  padding: 0px 4px;
  margin: 0px 3px;
  height: 100%;
  color: #f8f8f9;
  font-size: 12px;
  display: inline-block;
}
.main-footer-status {
  padding: 0px 4px;
  margin: 0px 3px;
  height: 100%;
  color: #515a6e;
  font-size: 12px;
  cursor: pointer;
  display: inline-block;
}
.main-footer-status:hover {
  background-color: #4BD2c0;
}
.main-footer-status-button {
  color: #f8f8f9;
}
</style>
