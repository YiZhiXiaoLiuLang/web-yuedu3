<template>
  <div>
    <!-- 域名输入弹窗 -->
    <el-dialog
      title="请输入域名"
      :visible.sync="domainDialogVisible"
      width="30%"
      :before-close="handleDomainDialogClose"
      :close-on-click-modal="false"
      :close-on-press-escape="false"
      :show-close="false"
    >
      <el-input style="display: none;" v-model="domain" placeholder="如有域名前缀，请输入（如 http://localhost:8088）。没有直接点确定"></el-input>
      <span slot="footer" class="dialog-footer">
        <el-button type="primary" @click="confirmDomain">load</el-button>
      </span>
    </el-dialog>

    <!-- 主内容区域 -->
    <div
      v-if="!domainDialogVisible"
      class="chapter-wrapper"
      :style="bodyTheme"
      :class="{ night: isNight, day: !isNight }"
    >
      <div class="tool-bar" :style="leftBarTheme">
        <div class="tools">
          <el-popover
            placement="right"
            :width="cataWidth"
            trigger="click"
            :visible-arrow="false"
            v-model="popCataVisible"
            popper-class="pop-cata"
          >
            <PopCata @getContent="getContent" ref="popCata" class="popup" />

            <div
              class="tool-icon"
              :class="{ 'no-point': noPoint }"
              slot="reference"
            >
              <div class="iconfont">
                &#58905;
              </div>
              <div class="icon-text">目录</div>
            </div>
          </el-popover>
          <el-popover
            placement="bottom-right"
            width="470"
            trigger="click"
            :visible-arrow="false"
            v-model="readSettingsVisible"
            popper-class="pop-setting"
          >
            <ReadSettings class="popup" />

            <div
              class="tool-icon"
              :class="{ 'no-point': noPoint }"
              slot="reference"
            >
              <div class="iconfont">
                &#58971;
              </div>
              <div class="icon-text">设置</div>
            </div>
          </el-popover>
          <div class="tool-icon" @click="toShelf">
            <div class="iconfont">
              &#58892;
            </div>
            <div class="icon-text">书架</div>
          </div>
          <div class="tool-icon" :class="{ 'no-point': noPoint }" @click="toTop">
            <div class="iconfont">
              &#58914;
            </div>
            <div class="icon-text">顶部</div>
          </div>
          <div
            class="tool-icon"
            :class="{ 'no-point': noPoint }"
            @click="toBottom"
          >
            <div class="iconfont">
              &#58915;
            </div>
            <div class="icon-text">底部</div>
          </div>
        </div>
      </div>
      <div class="read-bar" :style="rightBarTheme">
        <div class="tools">
          <div
            class="tool-icon"
            :class="{ 'no-point': noPoint }"
            @click="toLastChapter"
          >
            <div class="iconfont">
              &#58920;
            </div>
          </div>
          <div
            class="tool-icon"
            :class="{ 'no-point': noPoint }"
            @click="toNextChapter"
          >
            <div class="iconfont">
              &#58913;
            </div>
          </div>
        </div>
      </div>
      <div class="chapter-bar"></div>
      <div class="chapter" ref="content" :style="chapterTheme">
        <div class="content">
          <div class="top-bar" ref="top"></div>
          <div class="title" ref="title" v-if="show">{{ title }}</div>
          <Pcontent :carray="content" />
          <div class="bottom-bar" ref="bottom"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import PopCata from "../components/PopCatalog.vue";
import ReadSettings from "../components/ReadSettings.vue";
import Pcontent from "../components/Content.vue";
import Axios from "axios";
import jump from "../plugins/jump";
import config from "../plugins/config";

export default {
  components: {
    PopCata,
    Pcontent,
    ReadSettings
  },
  data() {
    return {
      domainDialogVisible: true, // 控制域名输入弹窗的显示
      domain: "", // 用户输入的域名
      title: "",
      content: [],
      noPoint: true,
      isNight: this.$store.state.config.theme == 6,
      bodyTheme: {
        background: config.themes[this.$store.state.config.theme].body
      },
      chapterTheme: {
        background: config.themes[this.$store.state.config.theme].content,
        width: this.$store.state.config.readWidth - 130 + "px"
      },
      leftBarTheme: {
        background: config.themes[this.$store.state.config.theme].popup,
        marginLeft: -(this.$store.state.config.readWidth / 2 + 68) + "px"
      },
      rightBarTheme: {
        background: config.themes[this.$store.state.config.theme].popup,
        marginRight: -(this.$store.state.config.readWidth / 2 + 52) + "px"
      }
    };
  },
  methods: {
    // 处理域名输入弹窗关闭
    handleDomainDialogClose(done) {
      if (!this.domain) {
        this.$message.warning("请输入域名");
        return;
      }
      done();
    },
    // 确认域名并加载内容
    confirmDomain() {
      //if (!this.domain) {
      //  this.$message.warning("请输入域名");
      //  return;
      //}
      this.domainDialogVisible = false; // 关闭弹窗
      this.loadContent(); // 加载内容
    },
    close_domain()
    {      this.domainDialogVisible = false; // Close the domain dialog
      this.loadContent(); // Load the content
    },
    // 加载内容
    loadContent() {
      this.loading = this.$loading({
        target: this.$refs.content,
        lock: true,
        text: "正在获取内容",
        spinner: "el-icon-loading",
        background: "rgba(0,0,0,0)"
      });

      // 从URL中获取书本ID
      const bookId = this.getBookIdFromUrl();
      if (!bookId) {
        this.$message.error("未找到书本ID");
        this.loading.close();
        return;
      }

      // 获取书本目录
      this.getCatalog(bookId).then(
        res => {
          let catalog = res.data;
          console.log(catalog);
          let book = {
            bookId: bookId,
            catalog: catalog
          };
          this.$store.commit("setReadingBook", book);

          // 默认加载第一章
          this.getContent(0);
          window.addEventListener('keyup', this.func_keyup);
        },
        err => {
          this.loading.close();
          this.$message.error("获取书籍目录失败");
          throw err;
        }
      );
    },
    // 从URL中获取书本ID
    getBookIdFromUrl() {
      const url = window.location.href;
	  const index = url.indexOf('?');
	  if (index !== -1) {
		return decodeURIComponent(url.slice(index + 1));
	  }
    },
    // 获取目录
    getCatalog(bookId) {
    console.log(`${bookId}/ChapterList.json`);
  return Axios.get(`${bookId}/ChapterList.json`)
    .then(res => {
      // 处理未自动解析的情况
      let rawData;
      try {
        rawData = typeof res.data === 'string' ? 
          JSON.parse(res.data) : 
          res.data;
      } catch (e) {
        console.error('JSON解析失败:', e);
        throw new Error("无效的JSON格式");
      }

      // 调试输出实际数据结构
      console.log('处理后的原始数据:', {
        type: typeof rawData,
        value: rawData
      });

      // 处理对象格式
      if (typeof rawData === 'object' && !Array.isArray(rawData)) {
        const catalogArray = Object.entries(rawData).map(([title, id]) => ({
          title,
          id: id // 确保转换为数字 //去你妈的转换尼玛呢
        })).sort((a, b) => a.id - b.id); // 按ID排序
        console.log("catalogArray",catalogArray);
        return { data: catalogArray };
      }

      // 处理其他格式
      throw new Error(`无法识别的数据结构: ${JSON.stringify(rawData)}`);
    })
    .catch(error => {
      console.error('获取目录失败:', error);
      throw error; // 保持错误传递
    });
},
    // 获取章节内容
    getContent(index) {
    console.log('getContent',index);
      this.$store.commit("setShowContent", false);
      
      this.$store.commit("setShowContent", false);
      
      this.$store.state.readingBook.index=index;
      if (!this.loading.visible) {
        this.loading = this.$loading({
          target: this.$refs.content,
          lock: true,
          text: "正在获取内容",
          spinner: "el-icon-loading",
          background: "rgba(0,0,0,0)"
        });
      }

      const bookId = this.getBookIdFromUrl();
      const chapterId = this.$store.state.readingBook.catalog[index].id;
	console.log(this.$store.state);
      //Axios.get(`${bookId}/${chapterId}.json`).then(
      console.log(`${bookId}/${chapterId}`);
      Axios.get(`${bookId}/${chapterId}`).then(
        res => {
          let data = res.data; // 访问data字段
          console.log(res);
          this.content = data.split(/\n+/);
          this.$store.commit("setContentLoading", true);
          this.loading.close();
          this.noPoint = false;
          this.$store.commit("setShowContent", true);
        },
        err => {
          this.$message.error("获取章节内容失败");
          this.content = "　　获取章节内容失败！";
          throw err;
        }
      );
    },
    toTop() {
      jump(this.$refs.top);
    },
    toBottom() {
      jump(this.$refs.bottom);
    },
    toNextChapter() {
      this.$store.commit("setContentLoading", true);
      let index = this.$store.state.readingBook.index;
      index++;
      console.log(index,this.$store.state.readingBook.catalog[index])
      if (typeof this.$store.state.readingBook.catalog[index] !== "undefined") {
        this.$message.info("下一章");
        this.getContent(index);
      } else {
        this.$message.error("本章是最后一章");
      }
    },
    toLastChapter() {
      this.$store.commit("setContentLoading", true);
      let index = this.$store.state.readingBook.index;
      index--;
      if (typeof this.$store.state.readingBook.catalog[index] !== "undefined") {
        this.$message.info("上一章");
        this.getContent(index);
      } else {
        this.$message.error("本章是第一章");
      }
    },
    toShelf() {
      this.$router.push("/");
    }
  }
};
</script>

<style lang="stylus" scoped>
>>>.el-dialog__wrapper {
  z-index: 9999 !important;
}

.chapter-wrapper {
  padding: 0 4%;
  flex-direction: column;
  align-items: center;

  >>>.no-point {
    pointer-events: none;
  }

  .tool-bar {
    position: fixed;
    top: 0;
    left: 50%;
    z-index: 100;

    .tools {
      display: flex;
      flex-direction: column;

      .tool-icon {
        font-size: 18px;
        width: 58px;
        height: 48px;
        text-align: center;
        padding-top: 12px;
        cursor: pointer;
        outline: none;

        .iconfont {
          font-family: iconfont;
          width: 16px;
          height: 16px;
          font-size: 16px;
          margin: 0 auto 6px;
        }

        .icon-text {
          font-size: 12px;
        }
      }
    }
  }

  .read-bar {
    position: fixed;
    bottom: 0;
    right: 50%;
    z-index: 100;

    .tools {
      display: flex;
      flex-direction: column;

      .tool-icon {
        font-size: 18px;
        width: 42px;
        height: 31px;
        padding-top: 12px;
        text-align: center;
        align-items: center;
        cursor: pointer;
        outline: none;
        margin-top: -1px;

        .iconfont {
          font-family: iconfont;
          width: 16px;
          height: 16px;
          font-size: 16px;
          margin: 0 auto 6px;
        }
      }
    }
  }

  .chapter-bar {
    .el-breadcrumb {
      .item {
        font-size: 14px;
        color: #606266;
      }
    }
  }

  .chapter {
    font-family: 'Microsoft YaHei', PingFangSC-Regular, HelveticaNeue-Light, 'Helvetica Neue Light', sans-serif;
    text-align: left;
    padding: 0 65px;
    min-height: 100vh;
    width: 670px;
    margin: 0 auto;

    >>>.el-icon-loading {
      font-size: 36px;
      color: #B5B5B5;
    }

    >>>.el-loading-text {
      font-weight: 500;
      color: #B5B5B5;
    }

    .content {
      font-size: 18px;
      line-height: 1.8;
      overflow: hidden;
      font-family: 'Microsoft YaHei', PingFangSC-Regular, HelveticaNeue-Light, 'Helvetica Neue Light', sans-serif;

      .title {
        margin-bottom: 57px;
        font: 24px / 32px PingFangSC-Regular, HelveticaNeue-Light, 'Helvetica Neue Light', 'Microsoft YaHei', sans-serif;
      }

      .bottom-bar, .top-bar {
        height: 64px;
      }
    }
  }
}

.day {
  >>>.popup {
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.12), 0 0 6px rgba(0, 0, 0, 0.04);
  }

  >>>.tool-icon {
    border: 1px solid rgba(0, 0, 0, 0.1);
    margin-top: -1px;
    color: #000;

    .icon-text {
      color: rgba(0, 0, 0, 0.4);
    }
  }

  >>>.chapter {
    border: 1px solid #d8d8d8;
    color: #262626;
  }
}

.night {
  >>>.popup {
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.48), 0 0 6px rgba(0, 0, 0, 0.16);
  }

  >>>.tool-icon {
    border: 1px solid #444;
    margin-top: -1px;
    color: #666;

    .icon-text {
      color: #666;
    }
  }

  >>>.chapter {
    border: 1px solid #444;
    color: #666;
  }

  >>>.popper__arrow {
    background: #666;
  }
}
</style>