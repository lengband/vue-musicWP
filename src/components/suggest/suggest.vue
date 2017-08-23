<template>
    <Scroll class="suggest" :data="result" @scrollToEnd="searchMore" :pullup="pullup" ref="suggest" @beforeScroll="listScroll" :beforeScroll="beforeScroll">
        <ul class="suggest-list">
            <li class="suggest-item" @click="selectItem(item)" v-for="(item, index) in result" :key="index" :ref="index === result.length - 1 ? 'liLast' : ''">
                <div class="icon">
                    <i :class="getIcnCls(item)"></i>
                </div>
                <div class="name">
                    <p class="text" v-html="getDisplayName(item)"></p>
                </div>
            </li>
            <Loading v-show="loadingShow" ref="loading"></Loading>
        </ul>
        <div class="no-result-wrapper" v-show="!hasMore && !result.length">
            <NoResult title="抱歉、暂无搜索结果"></NoResult>
        </div>
    </Scroll>
</template>

<script type="text/ecmascript-6">
import Scroll from 'base/scroll/scroll'
import Loading from 'base/loading/loading'
import NoResult from 'base/no-result/no-result'
import { search } from 'api/search'
import { ERR_OK } from 'api/config'
import { createSong } from 'common/js/song'
import { mapMutations, mapActions } from 'vuex'
import Singer from 'common/js/singer'
const TYPE_SINGER = 'singer'
const perpage = 20

export default {
    props: {
        query: {
            type: String,
            default: ''
        },
        showSinger: {
            type: Boolean,
            default: true
        }
    },
    data() {
        return {
            page: 1,
            result: [],
            pullup: true,
            hasMore: true, // 标志位 是否有更多的搜索内容
            loadingShow: false,
            beforeScroll: true
        }
    },
    methods: {
        search() {
            this.page = 1
            this.hasMore = true
            this.$refs.suggest.scrollTo(0, 0)
            search(this.query, this.page, this.showSinger, perpage).then((res) => {
                if (res.code === ERR_OK) {
                    this.result = this._genResult(res.data)
                    this._checkMore(res.data)
                }
            })
        },
        searchMore() {
            if (!this.hasMore) {
                return
            }
            this.page++
            this.loadingShow = true
            search(this.query, this.page, this.showSinger, perpage).then(res => {
                if (res.code === ERR_OK) {
                    this.result = this.result.concat(this._genResult(res.data))
                    this._checkMore(res.data)
                    this.$nextTick(() => {
                        this.loadingShow = false
                    })
                }
            })
        },
        getDisplayName(item) {
            if (item.type === TYPE_SINGER) {
                return item.singername
            } else {
                return `${item.name}-${item.singer}`
            }
        },
        getIcnCls(item) {
            if (item.type === TYPE_SINGER) {
                return 'icon-mine'
            } else {
                return 'icon-music'
            }
        },
        selectItem(item) {
            if (item.type === TYPE_SINGER) {
                const singer = new Singer({
                    id: item.singermid,
                    name: item.singername
                })
                this.$router.push({
                    path: `/search/${singer.id}`
                })
                this.setSinger(singer)
            } else {
                this.insertSong(item)
            }
            this.$emit('select')
        },
        refresh() {
            this.$refs.suggest.refresh()
        },
        listScroll() {
            this.$emit('listScroll')
        },
        _checkMore(data) {
            const song = data.song
            if (!song.list.length || (song.curnum + song.curpage * perpage) > song.totalnum) {
                this.hasMore = false
            }
        },
        _genResult(data) {
            let ret = []
            if (data.zhida && data.zhida.singerid) {
                ret.push({ ...data.zhida, ...{ type: TYPE_SINGER } })
            }
            if (data.song) {
                ret = ret.concat(this._normalizeSongs(data.song.list))
            }
            return ret
        },
        _normalizeSongs(list) {
            let ret = []
            list.forEach(musicDate => {
                if (musicDate.songid && musicDate.albumid) {
                    ret.push(createSong(musicDate))
                }
            })
            return ret
        },
        ...mapMutations({
            setSinger: 'SET_SINGER'
        }),
        ...mapActions([
            'insertSong'
        ])
    },
    watch: {
        query() {
            this.search()
        }
    },
    components: {
        Scroll,
        Loading,
        NoResult
    }
}
</script>

<style scoped lang="stylus" rel="stylesheet/stylus">
  @import "~common/stylus/variable"
  @import "~common/stylus/mixin"

  .suggest
    height: 100%
    overflow: hidden
    .suggest-list
      padding: 0 30px
      .suggest-item
        display: flex
        align-items: center
        padding-bottom: 20px
      .icon
        flex: 0 0 30px
        width: 30px
        [class^="icon-"]
          font-size: 14px
          color: $color-text-d
      .name
        flex: 1
        font-size: $font-size-medium
        color: $color-text-d
        overflow: hidden
        .text
          no-wrap()
    .no-result-wrapper
      position: absolute
      width: 100%
      top: 50%
      transform: translateY(-50%)
</style>