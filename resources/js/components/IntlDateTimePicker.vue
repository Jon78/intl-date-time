<template>
    <input :ref="refName"
           v-mask="maskFormat"
           :disabled="field.readonly"
           :dusk="field.attribute"
           :class="{'!cursor-not-allowed': field.readonly}"
           :value="value"
           :name="field.name"
           :placeholder="placeholder"
           type="text"
           :required="field.required || false"
    >
</template>

<script>
    import flatpickr                      from "flatpickr"
    import ShortcutButtonsPlugin          from "shortcut-buttons-flatpickr"
    import { mask }                       from "vue-the-mask"
    import { extend, localize, validate } from "vee-validate"
    import { Errors }                     from "laravel-nova"
    import { addDays, subDays }           from "date-fns"
    import DateTimeFormatConverter        from "../DateTimeFormatConverter"
    import { momentjsLocaleMapping }      from "../InternationalMapper"
    import locales                        from "../Locale"

    export default {

        directives: {
            mask,
        },
        props: {
            field: {
                type: Object,
                required: true,
            },
            value: {
                type: String,
                required: false,
                default: "",
            },
            readonly: {
                type: Boolean,
                default: false,
            },
            dateFormat: {
                type: String,
                default: "",
            },
            timeFormat: {
                type: String,
                default: "",
            },
            twelveHourTime: {
                type: Boolean,
                default: false,
            },
            enableTime: {
                type: Boolean,
                default: false,
            },
            enableSeconds: {
                type: Boolean,
                default: false,
            },
            locale: {
                type: String,
                default: "en-gb",
            },
            errorMessageLocale: {
                type: String,
                default: "en",
            },
            placeholder: {
                type: String,
                default: "",
            },
            minDate: {
                type: [Date, Object],
                default: null,
            },
            maxDate: {
                type: [Date, Object],
                default: null,
            },
        },

        data() {
            return {
                refName: "intlDatepickerInput",
                flatpickr: null,
                validationError: false,
                validationErrors: new Errors(),
                now: new Date(),
            }
        },

        computed: {
            momentjsFormat() {
                if (!this.timeFormat) {
                    if (this.field.displayLocaleTime) {
                        this.$set(this, "timeFormat", locales.momentjs[this.locale].LTS)
                    } else if (this.field.displayLocaleTimeShort) {
                        this.$set(this, "timeFormat", locales.momentjs[this.locale].LT)
                    }
                }

                return `${locales.momentjs[this.locale].L} ${this.timeFormat}`.replace(/[^ -~]+/g, "").trim()
            },

            dateFormatString() {
                if (this.dateFormat) {
                    return this.dateFormat
                }
                try {
                    return DateTimeFormatConverter.momentToFlatpickr(this.momentjsFormat)
                } catch (e) {
                    console.warn(e)
                }

                return "d/m/Y H:i:S"
            },

            maskFormat() {
                return this.momentjsFormat.replace(/\w/g, "#")
            },

            dateValidationRule() {
                return `date_format:${DateTimeFormatConverter.momentToDateFns(this.momentjsFormat)}`
            },

            displayShortcutButtons() {
                return this.field.displayShortcutButtons || false
            },

            required() {
                return this.field.required || false
            },
        },

        mounted() {
            import(/* webpackChunkName: "date_format_rule" */ "../validation/rules/date_format")
                .then((date_format) => {
                    extend("date_format", { ...date_format })

                    this.loadLocale(this.errorMessageLocale)
                })

            const shortcutButtons = [
                ...(this.displayShortcutButtons ? [
                    {
                        label: this.__("Yesterday"),
                    },
                    {
                        label: this.__("Today"),
                    },
                    {
                        label: this.__("Tomorrow"),
                    },
                ] : []),

                ...(this.required ? [] : [
                    {
                        label: this.__("Clear"),
                    }
                ]),
            ]

            const shortcutButtonFunctions = [
                ...(this.displayShortcutButtons ? [
                    (index, fp) => {
                        if (index > 2) {
                            return
                        }

                        let date
                        switch (index) {
                            case 0:
                                date = subDays(this.now, 1)
                                break
                            case 1:
                            default:
                                date = this.now
                                break
                            case 2:
                                date = addDays(this.now, 1)
                                break
                        }
                        fp.setDate(date)
                    }
                ] : []),

                ...(this.required ? [] : [
                    (index, fp) => {
                        if (index == 3 || !this.displayShortcutButtons) {
                            fp.clear()
                            fp.close()
                        }
                    },
                ]),
            ]

            const config = {
                enableTime: this.enableTime,
                enableSeconds: this.enableSeconds,
                onChange: this.onChange,
                onValueUpdate: this.onChange,
                onClose: this.onChange,
                dateFormat: this.dateFormatString,
                allowInput: true,
                time_24hr: true,
                locale: locales.flatpickr[momentjsLocaleMapping[this.locale].translation],
                plugins: shortcutButtons.length > 0 ? [
                    ShortcutButtonsPlugin({
                        button: shortcutButtons,
                        onClick: shortcutButtonFunctions,
                    }),
                ] : [],
            }

            if (this.minDate) {
                config.minDate = this.minDate.format(this.momentjsFormat)
            }

            if (this.maxDate) {
                config.maxDate = this.maxDate.format(this.momentjsFormat)
            }

            if (this.field.firstDayOfWeek !== undefined && !isNaN(Number(this.field.firstDayOfWeek))) {
                config.locale.firstDayOfWeek = Number(this.field.firstDayOfWeek)
            }

            this.$nextTick(() => {
                this.flatpickr = flatpickr(this.$refs[this.refName], config)
            })
        },

        methods: {
            onChange(selectedDates, dateStr) {
                if (dateStr) {
                    validate(dateStr, this.dateValidationRule, { name: this.field.name })
                        .then(({ valid, errors }) => {
                            if (valid) {
                                this.$set(this, "validationErrors", new Errors())
                                this.$set(this, "validationError", false)
                                this.$emit("change", dateStr)
                            } else {
                                this.$set(this, "validationErrors", new Errors(errors))
                                this.$set(this, "validationError", true)
                                this.$emit("error", this.validationErrors)
                            }
                        })
                } else {
                    this.$set(this, "validationErrors", new Errors())
                    this.$set(this, "validationError", false)
                    this.$emit("change", dateStr)
                }
            },

            loadLocale(code) {
                /**
                 * Asynchronously load the locale file then localize the validator with it
                 */
                import(/* webpackChunkName: "validation_locales/[request]" */ `vee-validate/dist/locale/${code}.json`)
                    .then((locale) => {
                        localize(code, locale)
                    })
                    .catch(() => {
                        console.warn(`The error messages do not support the '${code}' locale. Defaulting back to English. Please define another locale manually with errorMessageLocale().`)
                    })
            },
        },
    }
</script>

<style scoped>
    .\!cursor-not-allowed {
        cursor: not-allowed !important;
    }
</style>
