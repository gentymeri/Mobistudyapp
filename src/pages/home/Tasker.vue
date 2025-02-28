<template>
  <q-page padding>
    <q-pull-to-refresh
      @refresh="refresh"
      label="Default spinner"
    >
      <!-- Banner for when there is a new study available -->
      <q-banner
        rounded
        inline-actions
        class="bg-warning text-white q-mb-sm"
        v-if="newstudies"
        icon="new_releases"
        type="warning"
      >
        {{ $t('studies.newStudyAvailable') }}!
        <template v-slot:action>
          <q-btn
            color="blue"
            :label="$t('studies.checkNewStusy')"
            to="studies"
          />
        </template>
      </q-banner>
      <!-- end of banner -->

      <!-- Message for when there are not studies -->
      <q-item-label
        v-if="nostudies"
        class="q-title fixed-center"
      >
        {{ $t('studies.noStudies') }}
      </q-item-label>
      <!-- end of message -->

      <div
        v-else
        highlight
      >
        <!-- Pending tasks list -->
        <q-item-label header>
          {{ $t('studies.tasks.pendingTasks') }}
        </q-item-label>
        <q-list
          padding
          bordered
          v-for="study in studiesInfo"
          :key="`upcoming-${study._key}`"
          class="q-mt-sm"
        >
          <q-item-section>
            <q-item-label
              header
              overline
            >{{study.generalities.title[$root.$i18n.locale]}}</q-item-label>
            <taskListItem
              v-for="(task, index) in tasks.upcoming"
              :task="filterTask(task, study)"
              :isMissedTask="false"
              :key="`item-${index}`"
            >
            </taskListItem>
          </q-item-section>
          <q-item v-if="tasks.upcoming.length === 0">
            <q-item-section avatar>
              <q-icon
                color="primary"
                name="check"
              />
            </q-item-section>
            <q-item-section>
              <q-item-label>{{ $t('studies.tasks.noPendingTasks') }}</q-item-label>
            </q-item-section>
          </q-item>
        </q-list>
        <!-- end of pending tasks list -->

        <!-- Missed tasks list -->
        <q-item-label header>
          {{ $t('studies.tasks.missedTasks') }}
        </q-item-label>
        <q-list
          padding
          bordered
          v-for="study in studiesInfo"
          :key="`missed-${study._key}`"
          class="q-mt-sm"
        >
          <q-item-section v-if="tasks.missed.length > 0">
            <q-item-label
              header
              overline
            >{{study.generalities.title[$root.$i18n.locale]}}</q-item-label>
            <taskListItem
              v-for="(task, index) in tasks.missed"
              :task="filterTask(task, study)"
              :isMissedTask="true"
              :key="`item-${study._key}-${index}`"
            >
            </taskListItem>
          </q-item-section>
          <q-item v-else>
            <q-item-section avatar>
              <q-icon
                color="primary"
                name="check"
              />
            </q-item-section>
            <q-item-section>
              <q-item-label>{{ $t('studies.tasks.noMissedTasks') }}</q-item-label>
            </q-item-section>
          </q-item>
        </q-list>
        <!-- end of missed tasks list -->

        <!-- AlwaysOn tasks list -->
        <div v-if="tasks.alwaysOn.length > 0">
          <q-item-label header>
            {{ $t('studies.tasks.alwaysOnTasks') }}
          </q-item-label>
          <q-list
            padding
            bordered
            v-for="study in studiesInfo"
            :key="`alwayson-${study._key}`"
            class="q-mt-sm"
          >
            <q-item-section>
              <q-item-label
                header
                overline
              >{{study.generalities.title[$root.$i18n.locale]}}</q-item-label>
              <taskListItem
                v-for="(task, index) in tasks.alwaysOn"
                :task="filterTask(task, study)"
                :isMissedTask="false"
                :key="`item-${index}`"
              >
              </taskListItem>
            </q-item-section>
          </q-list>
        </div>
        <!-- end of alwaysOn tasks list -->

      </div>
    </q-pull-to-refresh>

    <!-- Completed study dialog -->
    <q-dialog
      v-if="this.tasks.completedStudyAlert"
      v-model="completedStudyModal"
      maximized
    >
      <div
        class="q-pa-lg text-center"
        style="background-color:white"
      >
        <div class="text-h4 q-mb-md">{{ $t('studies.studyCompletedHeadline') }}!</div>
        <div>
          <img
            src="imgs/confetti.svg"
            style="width:30vw; max-width:150px;"
          >
        </div>
        <p v-html="$t('studies.studyCompletedText', { studyname: completedStudyTitle })"></p>
        <q-btn
          color="primary"
          @click="studyCompleted()"
          :label="$t('common.close')"
        />
      </div>
    </q-dialog>
    <!-- end of completed study dialog -->

  </q-page>
</template>

<style>
</style>

<script>
import taskListItem from 'components/TaskListItem.vue'
import userinfo from 'modules/userinfo'
import DB from 'modules/db'
import API from 'modules/API/API'
import * as scheduler from 'modules/scheduler'

export default {
  name: 'TaskerPage',
  components: {
    taskListItem
  },
  data () {
    return {
      nostudies: false,
      newstudies: false,
      tasks: {
        upcoming: [],
        missed: [],
        alwaysOn: [],
        completedStudyAlert: undefined,
        completedStudyTitle: undefined
      },
      completedStudyModal: false,
      studiesInfo: []
    }
  },
  async created () {
    this.load()
    setTimeout(() => {
      this.load(false)
    }, 1000 * 60 * 60 * 12) // auto-reload every 12 hours
  },
  methods: {
    refresh (done) {
      this.load(true).then(done)
    },
    filterTask (task, study) {
      if (task.studyKey === study._key) {
        return task
      } else {
        return ''
      }
    },
    async load (skipSpinner) {
      if (!skipSpinner) this.$q.loading.show()
      try {
        // let's see if there are any new eligible studies
        try {
          let newStudyIds = await API.getNewStudiesKeys()
          if (newStudyIds.length > 0) {
            // there's a new study in town! warn the user!
            this.newstudies = true
          } else {
            this.newstudies = false
          }
        } catch (error) {
          console.error('Cannot connect to server, but thats OK', error)
          // if it fails it's fine
        }

        // the first time we show this component, tasks are re-scheduled
        await scheduler.cancelNotifications()
        try {
          // let's retrieve the studies from the API, just in case
          let profile = await API.getProfile(userinfo.user._key)
          if (!profile.studies || profile.studies.length === 0) {
            await DB.setStudiesParticipation([])
            // this user has no studies !
            this.$q.loading.hide()
            this.nostudies = true
            return
          } else {
            await DB.setStudiesParticipation(profile.studies)
          }
        } catch (error) {
          console.error('Cannot connect to server, but thats OK', error)
          // if it fails, we just rely on the stored data
        }

        // retrieve studies
        let studiesPart = await DB.getStudiesParticipation()
        if (!studiesPart) {
          // this user has no studies !
          this.$q.loading.hide()
          this.nostudies = true
          return
        }

        let activestudiesPart = []
        let activeStudiesDescr = []
        this.studiesInfo = []
        for (const studyPart of studiesPart) {
          let studyDescr = await DB.getStudyDescription(studyPart.studyKey)
          if (!studyDescr) {
            // study description needs to be retrieved from the server
            studyDescr = await API.getStudyDescription(studyPart.studyKey)
            await DB.setStudyDescription(studyPart.studyKey, studyDescr)
          }
          if (studyPart.currentStatus === 'accepted' && studyPart.reminders) {
            // schedule all of them
            await scheduler.scheduleNotificationsSingleStudy(studyDescr, studyPart)
          }
          if (studyPart.currentStatus === 'accepted') {
            activestudiesPart.push(studyPart)
            activeStudiesDescr.push(studyDescr)
            this.studiesInfo.push(studyDescr)
          }
        }

        if (activeStudiesDescr.length === 0) {
          // this user has no studies !
          this.$q.loading.hide()
          this.nostudies = true
          return
        }

        let response = scheduler.generateTasker(activestudiesPart, activeStudiesDescr)
        this.tasks = response

        if (response.completedStudyAlert) {
          this.completedStudyTitle = response.completedStudyAlert.studyTitle[this.$root.$i18n.locale]
          this.completedStudyModal = true
        }
        this.$q.loading.hide()
      } catch (error) {
        console.error(error)
        this.$q.loading.hide()

        this.$q.dialog({
          title: this.$i18n.t('errors.error'),
          message: this.$i18n.t('errors.generalError'),
          color: 'warning',
          ok: this.$i18n.t('common.retry'),
          preventClose: true
        }).onOk(() => {
          this.load()
        })
      }
    },
    async studyCompleted () {
      let studyPart = this.tasks.completedStudyAlert.studyPart
      // set the study as completed
      studyPart.currentStatus = 'completed'
      studyPart.completedTS = new Date()
      try {
        await API.updateStudyStatus(userinfo.user._key, studyPart.studyKey, studyPart)
        delete studyPart.extraItemsConsent
        delete studyPart.taskItemsConsent
        await DB.setStudyParticipation(studyPart)
        this.load()
        this.completedStudyModal = false
      } catch (error) {
        console.error('Cannot set the study as completed', error)
        this.$q.notify({
          color: 'negative',
          message: this.$i18n.t('errors.connectionError') + ': ' + error.message,
          icon: 'report_problem'
        })
      }
    }
  }
}
</script>
