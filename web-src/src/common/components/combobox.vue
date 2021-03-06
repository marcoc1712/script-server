<template>
    <div class="input-field combobox" :title="config.description" :data-error="error">
        <select
                :id="config.name"
                ref="selectField"
                class="validate"
                :required="config.required"
                :multiple="config.multiselect"
                :disabled="disabled || (options.length === 0)">
            <option :selected="!anythingSelected" value="" disabled>Choose your option</option>
            <option v-for="option in options"
                    :value="option.value"
                    :selected="option.selected">{{ option.value }}</option>
        </select>
        <label :for="config.name">{{ config.name }}</label>

        <ComboboxSearch ref="comboboxSearch" :comboboxWrapper="comboboxWrapper" v-if="searchEnabled"/>
    </div>
</template>

<script>
    import '@/common/materializecss/imports/select';
    import {addClass, contains, findNeighbour, isEmptyString, isNull, removeClass} from '@/common/utils/common';
    import {hasClass} from '../utils/common';
    import ComboboxSearch from './ComboboxSearch';

    export default {
        name: 'Combobox',
        components: {ComboboxSearch},
        props: {
            'config': Object,
            'value': [String, Array],
            'disabled': {
                type: Boolean,
                default: false
            },
            dropdownContainer: null
        },

        data: function () {
            return {
                options: [],
                anythingSelected: false,
                error: '',
                comboboxWrapper: null
            }
        },

        computed: {
            searchEnabled() {
                return !this.disabled && !this.config.multiselect && (this.options.length > 10);
            }
        },

        watch: {
            'config.values': {
                immediate: true,
                handler(newOptionValues) {
                    if (isNull(newOptionValues)) {
                        this.options = [];
                        return;
                    }

                    var newOptions = [];
                    for (var i = 0; i < newOptionValues.length; i++) {
                        newOptions.push({
                            value: newOptionValues[i],
                            selected: false
                        });
                    }

                    this.options = newOptions;

                    if (!this._fixValueByAllowedValues(this.config.values)) {
                        this._selectValue(this.value);
                    }

                    this.$nextTick(function () {
                        this.rebuildCombobox();
                    }.bind(this));
                }
            },

            'value': {
                immediate: true,
                handler(newValue) {
                    if (!this._fixValueByAllowedValues(this.config.values)) {
                        this._selectValue(newValue);
                    }
                }
            },

            disabled() {
                this.$nextTick(() => this.rebuildCombobox());
            }
        },

        mounted: function () {
            // for some reason subscription in template (i.e. @change="..." doesn't work for select input)
            this.$refs.selectField.onchange = () => {
                let value;
                if (this.config.multiselect) {
                    value = Array.from(this.$refs.selectField.selectedOptions)
                        .filter(o => !o.disabled)
                        .map(o => o.value)
                } else {
                    value = this.$refs.selectField.value;
                }
                this.emitValueChange(value);
            };

            this.rebuildCombobox();

            this.$watch('error', function (errorValue) {
                var inputField = findNeighbour(this.$refs.selectField, 'input');
                if (!isNull(inputField)) {
                    inputField.setCustomValidity(errorValue);
                }
            }, {
                immediate: true
            })
        },

        beforeDestroy: function () {
            const instance = M.FormSelect.getInstance(this.$refs.selectField);
            instance.destroy();
        },

        methods: {
            emitValueChange(value) {
                this._validate(this.asArray(value));
                this.$emit('input', value);
            },

            _fixValueByAllowedValues(allowedValues) {
                if (isNull(this.value) || (this.value === '') || (this.value === [])) {
                    return false;
                }

                var newValue;
                if (this.config.multiselect) {
                    if (!Array.isArray(this.value)) {
                        if (contains(allowedValues, this.value)) {
                            newValue = [this.value];
                        } else {
                            newValue = [];
                        }

                    } else {
                        newValue = [];
                        for (var i = 0; i < this.value.length; i++) {
                            var valueElement = this.value[i];
                            if (contains(allowedValues, valueElement)) {
                                newValue.push(valueElement)
                            }
                        }

                        if (newValue.length === this.value.length) {
                            return false;
                        }
                    }
                } else {
                    if (contains(allowedValues, this.value)) {
                        return false;
                    }

                    newValue = null;
                }

                this.emitValueChange(newValue);
                return true;
            },

            _selectValue(value) {
                const selectedValues = this.asArray(value);

                this.anythingSelected = false;

                const existingSelectedValues = new Set();

                for (var i = 0; i < this.options.length; i++) {
                    var option = this.options[i];
                    option.selected = contains(selectedValues, option.value);

                    if (option.selected) {
                        this.anythingSelected = true;
                        existingSelectedValues.add(option.value);
                    }
                }

                this._validate(selectedValues);

                this.$nextTick(function () {
                    this.updateComboboxValue();
                }.bind(this));
            },

            _validate(selectedValues) {
                if (this.config.required && (selectedValues.length === 0)) {
                    this.error = 'required';
                } else {
                    this.error = '';
                }
                this.$emit('error', this.error);
            },

            rebuildCombobox() {
                this.comboboxWrapper = M.FormSelect.init(this.$refs.selectField,
                    {
                        dropdownOptions: {
                            constrainWidth: false,
                            dropdownContainer: this.dropdownContainer
                        }
                    });

                const dropdownContent = this.$refs.selectField.parentNode.getElementsByClassName('dropdown-content')[0];

                const listItems = Array.from(dropdownContent.childNodes)
                    .filter(c => c.tagName === 'LI');

                listItems.forEach(item => {
                    const text = item.getElementsByTagName('span')[0].innerText;
                    if (text) {
                        item.title = text;
                    }
                });

                listItems.filter(item => hasClass(item, 'disabled'))
                    .forEach(headerItem => headerItem.setAttribute('tabindex', -1));


                this.updateComboboxValue();
            },

            updateComboboxValue() {
                if (isNull(this.$refs.selectField)) {
                    return;
                }

                const inputField = findNeighbour(this.$refs.selectField, 'input');

                // setCustomValidity doesn't work since input is readonly
                if (this.error) {
                    addClass(inputField, 'invalid');
                } else {
                    removeClass(inputField, 'invalid');
                }

                if (!isNull(this.comboboxWrapper)) {
                    this.comboboxWrapper._setValueToInput();
                    this.comboboxWrapper._setSelectedStates();
                }
            },

            asArray(value) {
                var valuesArray = value;

                if (isEmptyString(valuesArray)) {
                    valuesArray = [];
                } else if (!Array.isArray(valuesArray)) {
                    valuesArray = [valuesArray];
                }
                return valuesArray;
            }
        }
    }
</script>

<style scoped>
    .combobox >>> .search-hidden {
        display: none;
    }

    .main-app-content .combobox >>> .select-dropdown.dropdown-content .combobox-search-header {
        background-color: white;
        position: sticky;
        top: 0;
    }
</style>

