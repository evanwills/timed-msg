<template>
  <span :class="wrapClass" aria-live="polite">
    <span v-if="msgPrefix !== ''" class="timed-msg__prefix">{{ msgPrefix }}: </span>
    <span class="timed-msg__state">{{ humanPrefix }}{{ startWord }}</span>
    <span class="timed-msg__when" v-bind:key="lastUpdated" :title="cutoffDateTime">
      {{ dynamicDuration }}{{ endWord }}
    </span>
    <span v-if="msgSuffix !== ''" class="timed-msg__suffix">{{ msgSuffix }}</span>
  </span>
</template>


<script>
/**
 * Get a function that can be used within setTimout() to update
 * relevant strings;
 *
 * @param {VueComponent} context The `this` value for the current vue
 *                               component context
 *
 * @returns {function} A function that can be passed to setTimeout();
 */
const getStringUpdater = (context) => () => context.updateStrings();

/**
 * Get duration data based on the now time and the when time
 *
 * @param {number} now    Timestamp for current datetime
 *                        (usually from `Date.now()`)
 * @param {number} when   Timestamp for cut-off datetime
 *                        (usually from ISO 8601 datetime string)
 * @param {string} before Context text for message when rendered
 *                        before cut-off time
 * @param {string} after  Context text for message when rendered
 *                        after cut-off time
 * @param {string} end    Word/string to append to duration period
 *                        after cut-off period has passed
 *
 * @returns {object}
 */
const getDurationData = (now, when, before, after, start, end) => {
  /**
   * A list of the number of seconds per time unit
   *
   * Incase you're wondering about the two larger numbers:
   * * 1 year = 31557600 = (86400 * 365.25)
   * * 1 month = 2629800 = (31557600 / 12)
   *
   * @var {number[]} multipliers
   */
  const multipliers = [31557600, 2629800, 604800, 86400, 3600, 60];

  /**
   * List of time unit names
   *
   * @var {string[]} units
   */
  const units = ['year', 'month', 'week', 'day', 'hour', 'minute'];

  /**
   * Number of seconds difference between now and the cut-off time.
   *
   * @var {number} diff
   */
  let diff = Math.floor((when - now) / 1000);

  /**
   * Whether or not the `now` time is past the cut-off `when` time
   *
   * @var {boolean} isPast
   */
  const isPast = (diff < 0);

  /**
   * The number of seconds to set the next timeout duration
   *
   * @var {number} next
   */
  let next = 1;

  /**
   * The human value for the current duration
   *
   * @var {number} humanVal
   */
  let humanVal = 0;

  /**
   * The unit name for the current duration
   *
   * @var {string} unit
   */
  let unit = 'second';

  /**
   * Text to provide context to the duration rendered
   *
   * @property {string} humanPrefix
   */
  let prefixTxt = before;

  /**
   * The word to use at the end of the duration string
   *
   * If cut-off (`when`) has passed, `end` will be rendered
   *
   * @property {string} endWord
   */
  let endWord = '';
  let startWord = '';

  if (isPast === true) {
    diff *= -1;
    endWord = ` ${end}`;
    prefixTxt = after;
  } else {
    endWord = '';
    prefixTxt = before;
    startWord = ` ${start}`;
  }
  humanVal = diff;
  let extra = 0;

  // Find the best unit to render
  for (let a = 0; a < units.length; a += 1) {
    if (diff > multipliers[a]) {
      // This is the right unit to render

      extra = diff % multipliers[a];
      next = extra > 0 && isPast === false
        ? extra
        : multipliers[a];
      humanVal = diff / multipliers[a];
      unit = units[a];
      break;
    }
  }

  humanVal = Math.floor(humanVal);

  if (diff !== 0 && (next % 60) === 0) {
    // We need to do this to help the number tick over to the next
    // value otherwise, we'll have to wait a whole unit period.
    next = 1;
  }

  return {
    dynamicDuration: (diff === 0)
      ? 'now'
      : `${humanVal} ${unit}${(humanVal !== 1) ? 's' : ''}`,
    diff,
    endWord,
    isPast,
    prefixTxt,
    startWord: (diff === 0)
      ? ''
      : startWord,
    timeout: (next * 1000),
  };
};

export default {
  name: 'TimedMsg',

  props: {
    /**
     * Prefix string/word to use after the cut-off threshold has been
     * past
     *
     * @property {string} after
     */
    after: { type: String, required: false, default: 'Expired' },

    /**
     * Prefix string/word to use before the cut-off threshold has been
     * past
     *
     * @property {string} before
     */
    before: { type: String, required: false, default: 'Expires' },

    /**
     * End word to append to the duration to indicate the cut-off has
     * been passed
     *
     * @property {string} end
     */
    end: { type: String, required: false, default: 'ago' },

    /**
     * Prefix string/word to use before the cut-off threshold has been
     * past
     *
     * @property {string} before
     */
    id: { type: String, required: false, default: '' },

    /**
     * Any text to preced the duration being rendered
     *
     * @property {string} msgPrefix
     */
    msgPrefix: { type: String, required: false, default: '' },

    /**
     * Any text to follow the duration being rendered
     *
     * @property {string} msgPrefix
     */
    msgSuffix: { type: String, required: false, default: '' },

    noticeAt: { type: Number, required: false, default: -1 },

    /**
     * Start word to append to the `before` string to make the text
     * gramitcally correct before the cut-off has been passed
     *
     * @property {string} start
     */
    start: { type: String, required: false, default: 'in' },

    warnAt: { type: Number, required: false, default: -1 },

    /**
     * ISO 8601 date/time string for the cut-off time.
     *
     * @property {string} msgPrefix
     */
    when: { type: String, required: true },
  },

  data() {
    return {
      /**
       * The difference between the current time and when the cut-off
       * time occurs
       *
       * @property {number} diff
       */
      diff: 0,

      /**
       * The word to use at the end of the message string
       *
       * If cut-off (`this.when`) has passed, `endWord` will be
       * rendered
       *
       * @property {string} endWord
       */
      endWord: '',

      /**
       * Text to provide context to the time rendered
       *
       * @property {string} humanPrefix
       */
      humanPrefix: 'Expires',

      /**
       * The human readable duration (including unit) until/since the
       * cut-off time
       *
       * @property {string} dynamicDuration
       */
      dynamicDuration: '',

      /**
       * The Human readable time when cut-off occurs
       *
       * @property {string} cutoffDateTime
       */
      cutoffDateTime: '',

      /**
       * Whether or not the current time is past the cut-off time
       *
       * @property {boolean} isPast
       */
      isPast: false,

      /**
       * Timestamp for when the strings were last updated
       *
       * @property {number} lastUpdated
       */
      lastUpdated: 0,

      /**
       * ISO8601 datetime string for the cut-off.
       * Set when the component created or the `when` datetime string
       * was last changed)
       *
       * @property {string} oldWhen
       */
      oldWhen: '',

      /**
       * Start word to append to the `numanPrefix` string to make the
       * text gramitcally correct before the cut-off has been passed.
       *
       * @property {string} startWord
       */
      startWord: '',

      /**
       * Timestamp for the current cut-off datetime
       *
       * @property {number} lastUpdated
       */
      timestamp: 0,

      /**
       * current setTimeout() ID (used for clearing any timeouts)
       *
       * @property {number|null} timeout
       */
      timeouts: [],
    };
  },

  computed: {
    wrapClass() {
      const tmp = 'timed-msg';
      let output = tmp;

      if (this.isPast === true) {
        output += ` ${tmp}--expired`;
      } else if (this.warnAt > 0 && this.diff <= this.warnAt) {
        output += ` ${tmp}--warning`;
      } else if (this.noticeAt > 0 && this.diff <= this.noticeAt) {
        output += ` ${tmp}--notice`;
      }

      return output;
    },
  },

  methods: {
    updateStrings() {
      while (this.timeouts.length > 0) {
        const to = this.timeouts.shift()
        clearTimeout(to);
      }

      const tmpNow = Date.now();
      const tmp = getDurationData(
        tmpNow, this.timestamp,
        this.before, this.after,
        this.start, this.end,
      );

      const t = parseInt(this.id.substring(5));

      this.dynamicDuration = tmp.dynamicDuration;
      this.endWord = tmp.endWord;
      this.isPast = tmp.isPast;
      this.humanPrefix = tmp.prefixTxt;
      this.startWord = tmp.startWord;
      this.diff = tmp.diff;

      if (tmp.diff < 28800) { // 28800 = 8 hours
        this.timeouts.push(setTimeout(
          getStringUpdater(this),
          (tmp.timeout),
        ));
      }

      this.lastUpdated = tmpNow;
    },

    preset() {
      if (this.when !== this.oldWhen) {
        this.oldWhen = this.when;
        let tmp = null;

        try {
          tmp = new Date(this.when);
        } catch (error) {
          console.error(`Could not convert "${this.when}" to Date object`);
          return;
        }

        if (tmp.toString() === 'Invalid Date') {
          console.error(
            `Could not convert "${this.when}" to Date object because input is invalid`,
          );
        }

        const options = {
          weekday: 'long', year: 'numeric', month: 'long', day: 'numeric',
        };
        this.cutoffDateTime = `${tmp.toLocaleDateString(undefined, options)} `
                             + `at ${tmp.toLocaleTimeString('en-AU')}`;

        this.timestamp = tmp.getTime();
        this.updateStrings();
      }
    },
  },

  beforeMount() {
    this.preset();
  },

  updated() {
    this.preset();
  },
};
</script>

<style scoped>
.timed-msg {
  border-radius: 5rem;
  border: 0.05rem solid #fff;
  padding: 0.5rem 2rem;
  display: inline-block;
}
.timed-msg--expired {
  color: #aaa;
  border-color: #aaa;
}
.timed-msg--notice {
  color: #000;
  border-color: #000;
  background-color: yellow;
}
.timed-msg--warning {
  color: #fff;
  border-color: #fff;
  background-color: #800;
}
.timed-msg__when::before {
  content: ' ';
}
</style>
