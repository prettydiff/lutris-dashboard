<template>
  <div>
    <div v-if="submission">
      <h1>{{ submission.name }} ({{ submission.year }})
        <span style="float: right;">
          <button
            class="el-button el-button--primary el-button--small"
            @click="onToggleViewType">Toggle View</button>
        </span>
      </h1>

      <div>
        Submission for <strong>{{ submission.slug }}</strong> by {{ submission.user }} on {{ submittedAt }}
        <div v-if="originalInstaller" style="float: right;">
          <select
            id="revision-select"
            v-model="currentRevisionId"
            name="revisionSelect"
            @change="onRevisionSelect($event)">
            <option
              v-for="revision in originalInstaller.revisions"
              :key="revision.revision_id"
              :value="revision.revision_id">{{ revision.comment }}</option>
          </select>
        </div>
      </div>

      <p v-if="submission.draft">This submission is a draft, it may not be complete yet.</p>
      <p>{{ submission.reason }}</p>
      <div>
        <strong>Runner</strong>
        <div class="prettydiff" v-html="runnerDiff" />
      </div>
      <div>
        <strong>Version</strong>
        <div class="prettydiff" v-html="versionDiff" />
      </div>
      <div>
        <strong>Notes</strong>
        <div class="prettydiff" v-html="notesDiff" />
      </div>
      <div>
        <strong>Description</strong>
        <div class="prettydiff" v-html="descriptionDiff"/>
      </div>
      <div class="script-diff">
        <strong>Script</strong>
        <pre class="prettydiff" v-html="scriptDiff" />
        <button
          id="acceptSubmission"
          class="el-button el-button--primary el-button--small"
          @click="onSubmissionAccept">Accept</button>
        <button
          id="rejectSubmission"
          class="el-button el-button--danger el-button--small"
          @click="onRejectSubmission">Reject</button>
      </div>
    </div>
  </div>
</template>

<script>
import { fetchSubmission, fetchRevisions, acceptSubmission, deleteSubmission } from '@/api/installers'
import { Message } from 'element-ui'
import prettydiff from 'prettydiff'
import moment from 'moment'
export default {
  name: 'SubmissionDetail',
  data() {
    return {
      submission: null,
      installers: null,
      originalInstaller: null,
      currentRevisionId: null,
      submissionLoading: false,
      revisionsLoading: false,
      contentDiff: null,
      viewType: 'inline'
    }
  },
  computed: {
    revisionId() {
      return this.$route.params && this.$route.params.id
    },
    submittedAt() {
      const created_moment = moment(this.submission.created_at)
      return created_moment.format('MMMM Do hh:mm') + ' (' + created_moment.fromNow() + ')'
    },
    runnerDiff() {
      if (this.originalInstaller) {
        return this.outputDiff(this.originalInstaller.runner, this.submission.runner, this.viewType)
      }
      return this.submission.runner
    },
    versionDiff() {
      if (this.originalInstaller) {
        return this.outputDiff(this.originalInstaller.version, this.submission.version, this.viewType)
      }
      return this.submission.version
    },
    descriptionDiff() {
      if (this.originalInstaller) {
        return this.outputDiff(this.originalInstaller.description, this.submission.description, this.viewType)
      }
      return this.submission.notes
    },
    notesDiff() {
      if (this.originalInstaller) {
        return this.outputDiff(this.originalInstaller.notes, this.submission.notes, this.viewType)
      }
      return this.submission.notes
    },
    scriptDiff() {
      if (this.originalInstaller) {
        return this.outputDiff(this.originalInstaller.content, this.submission.content, this.viewType)
      }
      return this.submission.content
    }
  },
  created() {
    this.getSubmission()
    this.currentRevisionId = this.revisionId
  },
  methods: {
    outputDiff(originalText, newText, viewType) {
      if (!originalText) {
        originalText = ''
      }
      if (!newText) {
        newText = ''
      }
      originalText = originalText.replace('\r\n', '\n')
      newText = newText.replace('\r\n', '\n')
      const options = prettydiff.defaults
      options.mode = 'diff'
      options.language = 'text'
      options.diff_format = 'html'
      options.diff_space_ignore = false
      options.diff_view = viewType
      options.source = originalText
      options.diff = newText
      return prettydiff.mode(options)
    },

    getSubmission() {
      this.submissionLoading = true
      fetchSubmission(this.revisionId).then(response => {
        this.submission = response.data
        this.submissionsLoading = false
        this.getRevisions(this.submission.game_slug)
      })
    },
    getRevisions(slug) {
      this.revisionsLoading = true
      fetchRevisions(slug).then(response => {
        this.revisionsLoading = false
        this.installers = response.data.installers
        for (const installer of this.installers) {
          if (this.submission.installer_id === installer.id) {
            this.originalInstaller = installer
            break
          }
        }
      })
    },
    onSubmissionAccept() {
      acceptSubmission(this.revisionId).then(response => {
        Message({
          message: 'Submission has been accepted',
          type: 'info',
          duration: 5000
        })
        this.$router.push({ path: '/installers/submissions' })
      })
    },
    onRejectSubmission() {
      deleteSubmission(this.revisionId).then(response => {
        Message({
          message: 'Submission has been deleted',
          type: 'info',
          duration: 5000
        })
        this.$router.push({ path: '/installers/submissions' })
      })
    },
    onToggleViewType() {
      if (this.viewType === 'inline') {
        this.viewType = 'sidebyside'
      } else {
        this.viewType = 'inline'
      }
    },
    onRevisionSelect(event) {
      for (const revision of this.originalInstaller.revisions) {
        if (revision.revision_id === this.currentRevisionId) {
          this.submission = revision
        }
      }
    }
  }
}
</script>

<style lang="scss">
.prettydiff > p {
  display: none;
}
.diff {
  background-color: #E4F1FE;
  display: flex;
  font-family: 'Courier New', Courier, monospace;
  padding: 0 1em;
  ol {
    padding: 0;
    vertical-align: top;
    display: inline-block;
    list-style-type: none;
  }
  ol.count {
    display: none;
  }
  li {
    line-height: 1.4em;
  }
  h3.texttitle, p.author {
    display: none;
  }
  .delete {
    background-color: #FFCFF7;
  }
  .equal {
    background-color: #E4F1FE;
  }
  .insert {
    background-color: #C4FCDC;
  }
  .empty {
    line-height: 1em;
    height: 1em;
    background-color: #FFCFF7;
  }
  em {
    outline: 1px dotted salmon;
    background-color: #FFC46C;
    line-height: 0.8em;
    padding: 0;
  }
  .replace {
    background-color: #FFEED5;
  }
  .diff-left {
    border-right: 2px solid #AAA;
  }
  .diff-left, .diff-right {
    padding: 0 0.5em;
    width: 50%;
    overflow: scroll;
  }
}
</style>
