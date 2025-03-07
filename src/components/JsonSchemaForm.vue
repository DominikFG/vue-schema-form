<template>
    <div class="json-schema-form">
        <h2 v-if="schema.title">{{ schema.title }}</h2>
        <p v-if="schema.description" class="schema-description">{{ schema.description }}</p>

        <form @submit.prevent="handleSubmit">
            <template v-if="schema.type === 'object'">
                <div v-for="(property, key) in schema.properties" :key="key" class="form-field"
                    :class="{ 'nested-field': level > 0 }">
                    <div v-if="property.type !== 'object' && property.type !== 'array'" class="field-container">
                        <label :for="`${fieldPrefix}${key}`">{{ property.title || key }}{{
                            schema.required?.includes(key) ? ' *' : '' }}</label>

                        <div v-if="property.description" class="field-description">{{ property.description }}</div>

                        <!-- String type -->
                        <input v-if="property.type === 'string' && !property.enum && !property.format"
                            :id="`${fieldPrefix}${key}`" v-model="formData[key]" type="text"
                            :required="schema.required?.includes(key)" :placeholder="property.examples?.[0] || ''" />

                        <!-- Number type -->
                        <input v-else-if="property.type === 'number' || property.type === 'integer'"
                            :id="`${fieldPrefix}${key}`" v-model.number="formData[key]" type="number"
                            :step="property.type === 'integer' ? 1 : 'any'" :min="property.minimum"
                            :max="property.maximum" :required="schema.required?.includes(key)" />

                        <!-- Boolean type -->
                        <div v-else-if="property.type === 'boolean'" class="checkbox-wrapper">
                            <input :id="`${fieldPrefix}${key}`" v-model="formData[key]" type="checkbox" />
                        </div>

                        <!-- Enum (select) -->
                        <select v-else-if="property.enum" :id="`${fieldPrefix}${key}`" v-model="formData[key]"
                            :required="schema.required?.includes(key)">
                            <option value="" disabled>-- Select an option --</option>
                            <option v-for="option in property.enum" :key="option" :value="option">
                                {{ property.enumNames?.[property.enum.indexOf(option)] || option }}
                            </option>
                        </select>

                        <!-- Email format -->
                        <input v-else-if="property.format === 'email'" :id="`${fieldPrefix}${key}`"
                            v-model="formData[key]" type="email" :required="schema.required?.includes(key)" />

                        <!-- Date format -->
                        <input v-else-if="property.format === 'date'" :id="`${fieldPrefix}${key}`"
                            v-model="formData[key]" type="date" :required="schema.required?.includes(key)" />

                        <!-- Textarea for larger string inputs -->
                        <textarea v-else-if="property.format === 'textarea'" :id="`${fieldPrefix}${key}`"
                            v-model="formData[key]" :rows="property.rows || 4"
                            :required="schema.required?.includes(key)"></textarea>

                        <!-- Default to text input for any other types -->
                        <input v-else :id="`${fieldPrefix}${key}`" v-model="formData[key]" type="text"
                            :required="schema.required?.includes(key)" />

                        <div v-if="errors[key]" class="field-error">{{ errors[key] }}</div>
                    </div>

                    <!-- Nested object -->
                    <div v-else-if="property.type === 'object'" class="nested-object">
                        <h3>{{ property.title || key }}</h3>
                        <div v-if="property.description" class="field-description">{{ property.description }}</div>

                        <JsonSchemaForm :schema="property" :modelValue="formData[key] || {}"
                            @update:modelValue="updateNestedValue(key, $event)" :errors="getNestedErrors(key)"
                            :field-prefix="`${fieldPrefix}${key}.`" :level="level + 1" />
                    </div>

                    <!-- Array type -->
                    <div v-else-if="property.type === 'array'" class="array-field">
                        <h3>{{ property.title || key }}</h3>
                        <div v-if="property.description" class="field-description">{{ property.description }}</div>

                        <!-- Array of primitive values -->
                        <div v-if="property.items && ['string', 'number', 'integer', 'boolean'].includes(property.items.type)"
                            class="array-primitives">
                            <div v-for="(_, index) in formData[key] || []" :key="`${key}_${index}`" class="array-item">
                                <!-- String item -->
                                <input
                                    v-if="property.items && property.items.type === 'string' && !property.items.enum && !property.items.format"
                                    v-model="formData[key][index]" type="text"
                                    :placeholder="property.items?.examples?.[0] || ''" />

                                <!-- Number item -->
                                <input
                                    v-else-if="property.items && (property.items.type === 'number' || property.items.type === 'integer')"
                                    v-model.number="formData[key][index]" type="number"
                                    :step="property.items?.type === 'integer' ? 1 : 'any'"
                                    :min="property.items?.minimum" :max="property.items?.maximum" />

                                <!-- Boolean item -->
                                <div v-else-if="property.items && property.items.type === 'boolean'"
                                    class="checkbox-wrapper">
                                    <input v-model="formData[key][index]" type="checkbox" />
                                </div>

                                <!-- Enum item -->
                                <select v-else-if="property.items && property.items.enum"
                                    v-model="formData[key][index]">
                                    <option value="" disabled>-- Select an option --</option>
                                    <option v-for="option in property.items.enum" :key="option" :value="option">
                                        {{ property.items.enumNames?.[property.items.enum.indexOf(option)] || option }}
                                    </option>
                                </select>

                                <button type="button" class="array-remove-btn" @click="removeArrayItem(key, index)">
                                    Remove
                                </button>
                            </div>

                            <button type="button" class="array-add-btn"
                                @click="property.items && addArrayItem(key, property.items.type)">
                                Add Item
                            </button>
                        </div>

                        <!-- Array of objects -->
                        <div v-else-if="property.items && property.items.type === 'object'" class="array-objects">
                            <div v-for="(item, index) in formData[key] || []" :key="`${key}_${index}`"
                                class="array-object-item">
                                <div class="array-item-header">
                                    <h4>Item {{ index + 1 }}</h4>
                                    <button type="button" class="array-remove-btn" @click="removeArrayItem(key, index)">
                                        Remove
                                    </button>
                                </div>

                                <JsonSchemaForm :schema="property.items as JsonSchema" :modelValue="item"
                                    @update:modelValue="updateArrayObjectItem(key, index, $event)"
                                    :errors="getNestedArrayErrors(key, index)"
                                    :field-prefix="`${fieldPrefix}${key}[${index}].`" :level="level + 1" />
                            </div>

                            <button type="button" class="array-add-btn"
                                @click="property.items && addArrayObjectItem(key, property.items)">
                                Add Item
                            </button>
                        </div>
                    </div>
                </div>
            </template>

            <div v-if="level === 0" class="form-actions">
                <button type="submit" :disabled="isSubmitting">{{ submitButtonText }}</button>
            </div>
        </form>
    </div>
</template>

<script lang="ts">
import { defineComponent, ref, reactive, computed, watch } from 'vue';
import type { PropType } from 'vue';

// Define TypeScript interfaces
interface JsonSchema {
    title?: string;
    description?: string;
    type: string;
    properties?: Record<string, SchemaProperty>;
    required?: string[];
    items?: SchemaProperty;
}

interface SchemaProperty {
    title?: string;
    description?: string;
    type: string;
    default?: any;
    enum?: any[];
    enumNames?: string[];
    format?: string;
    minimum?: number;
    maximum?: number;
    minLength?: number;
    maxLength?: number;
    pattern?: string;
    examples?: any[];
    rows?: number;
    properties?: Record<string, SchemaProperty>;
    required?: string[];
    items?: SchemaProperty;
    minItems?: number;
    maxItems?: number;
}

type FormData = Record<string, any>;
type ErrorObject = Record<string, string>;

export default defineComponent({
    name: 'JsonSchemaForm',

    props: {
        schema: {
            type: Object as PropType<JsonSchema>,
            required: true
        },
        modelValue: {
            type: Object as PropType<FormData>,
            default: () => ({})
        },
        initialData: {
            type: Object as PropType<FormData>,
            default: () => ({})
        },
        submitButtonText: {
            type: String,
            default: 'Submit'
        },
        errors: {
            type: Object as PropType<ErrorObject>,
            default: () => ({})
        },
        fieldPrefix: {
            type: String,
            default: ''
        },
        level: {
            type: Number,
            default: 0
        }
    },

    emits: ['submit', 'validation-error', 'update:modelValue'],

    setup(props, { emit }) {
        const isSubmitting = ref < boolean > (false);
        const localErrors = reactive < ErrorObject > ({});

        // Initialize form data with schema defaults and any initial data
        const formData = reactive < FormData > ({});

        // Watch for modelValue changes (for nested components)
        watch(() => props.modelValue, (newValue) => {
            // Update formData with modelValue
            Object.keys(formData).forEach(key => {
                delete formData[key];
            });

            Object.entries(newValue).forEach(([key, value]) => {
                formData[key] = value;
            });
        }, { deep: true, immediate: true });

        // Watch formData changes to emit updates
        watch(formData, (newValue) => {
            if (props.level > 0) {
                emit('update:modelValue', { ...newValue });
            }
        }, { deep: true });

        // Initialize form data with defaults and initial values
        const initializeFormData = (): void => {
            // Set default values from schema
            if (props.schema.properties) {
                Object.entries(props.schema.properties).forEach(([key, property]: [string, SchemaProperty]) => {
                    // If we have a value in modelValue, use that
                    if (props.modelValue[key] !== undefined) {
                        return;
                    }

                    // Otherwise check initialData for top-level form
                    if (props.level === 0 && props.initialData[key] !== undefined) {
                        formData[key] = props.initialData[key];
                        return;
                    }

                    // Otherwise use schema default or create empty value
                    if (property.default !== undefined) {
                        formData[key] = property.default;
                    } else {
                        // Set empty values based on type
                        switch (property.type) {
                            case 'string':
                                formData[key] = '';
                                break;
                            case 'number':
                            case 'integer':
                                formData[key] = null;
                                break;
                            case 'boolean':
                                formData[key] = false;
                                break;
                            case 'array':
                                formData[key] = [];
                                break;
                            case 'object':
                                formData[key] = {};
                                break;
                            default:
                                formData[key] = null;
                        }
                    }
                });
            }
        };

        // Initialize the form data when the component is created
        initializeFormData();

        // Get default value for array items based on type
        const getDefaultForType = (type: string): any => {
            switch (type) {
                case 'string': return '';
                case 'number':
                case 'integer': return 0;
                case 'boolean': return false;
                case 'object': return {};
                case 'array': return [];
                default: return null;
            }
        };

        // Add item to array
        const addArrayItem = (key: string, type: string): void => {
            if (!formData[key]) {
                formData[key] = [];
            }
            formData[key].push(getDefaultForType(type));
        };

        // Add object item to array
        const addArrayObjectItem = (key: string, schema: SchemaProperty): void => {
            if (!formData[key]) {
                formData[key] = [];
            }

            // Create a new object with default values for each property
            const newItem: Record<string, any> = {};
            if (schema.properties) {
                Object.entries(schema.properties).forEach(([propKey, property]) => {
                    if (property.default !== undefined) {
                        newItem[propKey] = property.default;
                    } else {
                        newItem[propKey] = getDefaultForType(property.type);
                    }
                });
            }

            formData[key].push(newItem);
        };

        // Remove item from array
        const removeArrayItem = (key: string, index: number): void => {
            formData[key].splice(index, 1);
        };

        // Update nested value (for object properties)
        const updateNestedValue = (key: string, value: any): void => {
            formData[key] = value;
        };

        // Update array object item
        const updateArrayObjectItem = (key: string, index: number, value: any): void => {
            formData[key][index] = value;
        };

        // Get nested errors (for object properties)
        const getNestedErrors = (key: string): ErrorObject => {
            const result: ErrorObject = {};

            // Check if there are errors for nested properties
            Object.entries(props.errors).forEach(([errorKey, errorValue]) => {
                if (errorKey.startsWith(`${key}.`)) {
                    const nestedKey = errorKey.substring(key.length + 1);
                    result[nestedKey] = errorValue;
                }
            });

            return result;
        };

        // Get nested array errors
        const getNestedArrayErrors = (key: string, index: number): ErrorObject => {
            const result: ErrorObject = {};
            const prefix = `${key}[${index}].`;

            // Check if there are errors for nested array items
            Object.entries(props.errors).forEach(([errorKey, errorValue]) => {
                if (errorKey.startsWith(prefix)) {
                    const nestedKey = errorKey.substring(prefix.length);
                    result[nestedKey] = errorValue;
                }
            });

            return result;
        };

        // Validate form data against schema
        const validateForm = (data: FormData = formData, schema: JsonSchema = props.schema, path: string = ''): ErrorObject => {
            const errors: ErrorObject = {};

            // Check required fields
            if (schema.required && schema.properties) {
                for (const field of schema.required) {
                    if (data[field] === undefined || data[field] === null || data[field] === '') {
                        errors[`${path}${field}`] = `${schema.properties[field]?.title || field} is required`;
                    }
                }
            }

            // Validate each property
            Object.entries(schema.properties || {}).forEach(([key, property]) => {
                const value = data[key];
                const fullPath = path ? `${path}${key}` : key;

                if (value !== undefined && value !== null && value !== '') {
                    // Type validation for primitive types
                    if (property.type === 'number' || property.type === 'integer') {
                        if (isNaN(Number(value))) {
                            errors[fullPath] = `${property.title || key} must be a valid number`;
                        } else {
                            if (property.type === 'integer' && !Number.isInteger(Number(value))) {
                                errors[fullPath] = `${property.title || key} must be an integer`;
                            }

                            if (property.minimum !== undefined && value < property.minimum) {
                                errors[fullPath] = `${property.title || key} must be at least ${property.minimum}`;
                            }

                            if (property.maximum !== undefined && value > property.maximum) {
                                errors[fullPath] = `${property.title || key} must be at most ${property.maximum}`;
                            }
                        }
                    }

                    // String validation
                    if (property.type === 'string') {
                        if (property.minLength !== undefined && value.length < property.minLength) {
                            errors[fullPath] = `${property.title || key} must be at least ${property.minLength} characters`;
                        }

                        if (property.maxLength !== undefined && value.length > property.maxLength) {
                            errors[fullPath] = `${property.title || key} must be at most ${property.maxLength} characters`;
                        }

                        if (property.pattern) {
                            const regex = new RegExp(property.pattern);
                            if (!regex.test(value)) {
                                errors[fullPath] = `${property.title || key} is not in the correct format`;
                            }
                        }

                        // Format validation
                        if (property.format === 'email') {
                            const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
                            if (!emailRegex.test(value)) {
                                errors[fullPath] = `${property.title || key} must be a valid email address`;
                            }
                        }
                    }

                    // Enum validation
                    if (property.enum && !property.enum.includes(value)) {
                        errors[fullPath] = `${property.title || key} must be one of the allowed values`;
                    }

                    // Object validation
                    if (property.type === 'object') {
                        const nestedErrors = validateForm(value, property, `${fullPath}.`);
                        Object.assign(errors, nestedErrors);
                    }

                    // Array validation
                    if (property.type === 'array' && property.items) {
                        // Check minItems and maxItems
                        if (property.minItems !== undefined && value.length < property.minItems) {
                            errors[fullPath] = `${property.title || key} must have at least ${property.minItems} items`;
                        }

                        if (property.maxItems !== undefined && value.length > property.maxItems) {
                            errors[fullPath] = `${property.title || key} must have at most ${property.maxItems} items`;
                        }

                        // Validate each item in the array
                        if (Array.isArray(value)) {
                            value.forEach((item, index) => {
                                if (property.items && property.items.type === 'object') {
                                    const nestedErrors = validateForm(item, property.items, `${fullPath}[${index}].`);
                                    Object.assign(errors, nestedErrors);
                                }
                            });
                        }
                    }
                }
            });

            return errors;
        };

        const handleSubmit = (): void => {
            if (props.level > 0) {
                return; // Only the top-level form can be submitted
            }

            const validationErrors = validateForm();

            if (Object.keys(validationErrors).length === 0) {
                isSubmitting.value = true;
                emit('submit', { ...formData });

                // Reset submission state after a delay
                setTimeout(() => {
                    isSubmitting.value = false;
                }, 500);
            } else {
                // Update local errors
                Object.entries(validationErrors).forEach(([key, value]) => {
                    localErrors[key] = value;
                });

                emit('validation-error', validationErrors);
            }
        };

        return {
            formData,
            errors: computed(() => {
                // For top-level form, use local errors; for nested forms, use props.errors
                return props.level === 0 ? localErrors : props.errors;
            }),
            isSubmitting,
            handleSubmit,
            addArrayItem,
            removeArrayItem,
            addArrayObjectItem,
            updateNestedValue,
            updateArrayObjectItem,
            getNestedErrors,
            getNestedArrayErrors
        };
    }
});
</script>

<style>
.json-schema-form {
    font-family: sans-serif;
    max-width: 800px;
    margin: 0 auto;
}

.schema-description {
    margin-bottom: 20px;
    color: #666;
}

.form-field {
    margin-bottom: 20px;
}

.nested-field {
    margin-left: 20px;
    padding-left: 10px;
    border-left: 1px solid #eee;
}

.nested-object,
.array-field {
    margin-bottom: 20px;
    padding: 15px;
    background-color: #f9f9f9;
    border-radius: 4px;
}

.array-item {
    display: flex;
    align-items: center;
    margin-bottom: 10px;
}

.array-item>*:first-child {
    flex: 1;
    margin-right: 10px;
}

.array-object-item {
    margin-bottom: 20px;
    padding: 15px;
    background-color: #f0f0f0;
    border-radius: 4px;
}

.array-item-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 10px;
}

.array-item-header h4 {
    margin: 0;
}

.array-add-btn,
.array-remove-btn {
    padding: 5px 10px;
    font-size: 0.85em;
    background-color: #e0e0e0;
    border: none;
    border-radius: 4px;
    cursor: pointer;
}

.array-add-btn {
    margin-top: 10px;
    background-color: #4caf50;
    color: white;
}

.array-remove-btn {
    background-color: #f44336;
    color: white;
}

label {
    display: block;
    font-weight: bold;
    margin-bottom: 5px;
}

.field-description {
    font-size: 0.85em;
    color: #666;
    margin-bottom: 5px;
}

input[type="text"],
input[type="number"],
input[type="email"],
input[type="date"],
select,
textarea {
    width: 100%;
    padding: 8px;
    border: 1px solid #ddd;
    border-radius: 4px;
    box-sizing: border-box;
    font-size: 1em;
}

.checkbox-wrapper {
    padding: 8px 0;
}

input[type="checkbox"] {
    margin-right: 8px;
}

.field-error {
    color: #d32f2f;
    font-size: 0.85em;
    margin-top: 5px;
}

.form-actions {
    margin-top: 30px;
}

button[type="submit"] {
    background-color: #4caf50;
    color: white;
    border: none;
    padding: 10px 15px;
    font-size: 1em;
    border-radius: 4px;
    cursor: pointer;
}

button[type="submit"]:hover {
    background-color: #45a049;
}

button[type="submit"]:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
}
</style>