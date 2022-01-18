<template>
  <div class="container">
    <h1>Plain Podcast Player <span class="pull-right"><player :track="selectedTrack"/></span></h1>
    <hr>

    <div class="row">
      <feedslist :feeds="feeds" :selected-feed-title="selectedFeed.title" @on-feed-select="feedSelected"/>
      <feedview :feed="selectedFeed" :selected-track="selectedTrack" @on-track-select="trackSelected"/>
    </div>
  </div>
</template>

<script>
import { parseURL } from 'rss-parser'

import feedslist from '@/components/feedslist.vue'
import feedview from '@/components/feedview.vue'
import player from '@/components/player.vue'

import * as WebLN from 'webln'
import axios from 'axios';
import crypto from 'crypto';

export default {
  components: { feedslist, feedview, player },

  data() {
    return {
      feeds: {},
      selectedFeed: {},
      selectedTrack: ""
    }
  },

  methods: {
    feedSelected (name) {
      if (!this.feeds[name]) this.selectedFeed = {}
      this.selectedFeed = this.feeds[name]
    },

    async trackSelected (url) {
      await this.pay(url);
    },
    async fetchInvoice(amountInMsats) {
      const invoice = await axios.get(this.selectedFeed.addressCallback, {params: {amount: amountInMsats}})
      return invoice.data.pr;
    },
    async pay(url) {
      // 1. check if webln is available
      if (!window.webln) {
        alert("WebLN not available");
        return;
      }
      const amount = 100;
      const amountInMsats = parseInt(amount) * 1000; // we need msats
  
      // get a lightning invoice
      this.fetchInvoice(amountInMsats).then((invoice) => {
        console.log(`Paying invoice ${invoice}`);

        // 2. enable webln - prompts the user for the first time and asks for permission
        webln.enable().then(() => {

          // 3. pay the invoice
          webln.sendPayment(invoice).then((response) => {
              if (!url) this.selectedTrack = ""
              this.selectedTrack = url
          }).catch((e) => {
            alert("Cancelled... here we should show a fallback e.g. a QR code");
            return false;
          });
        }).catch((e) => {
           alert("Cancelled... here we should show a fallback e.g. a QR code");
           return false;
        });
      });
      return false;
    }
  },

  mounted() {
    fetch("/static/feeds.json")
      .then(res => res.json())
      .then(res => {
        if (res.length) {
          res.forEach(feed => {
            parseURL(feed.url, (err, data) => {
              if (err) console.error(err)
              const feedData = data.feed;
              feedData.addressCallback = feed.addressPayload.callback;
              this.$set(this.feeds, data.feed.title, feedData)
            })

          })
        } else {
          alert("No Podcast urls found")
        }
      })

  }
}
</script>