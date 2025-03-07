# Vue JSON Schema Form

A Vue 3 component that automatically generates forms from JSON Schema definitions. This component supports various input types, nested objects, arrays, and validation according to the JSON Schema specification.

## Features
🧩 Dynamic form generation based on JSON Schema
📝 Supports multiple input types (text, number, boolean, select, email, date, textarea)
📋 Handles nested objects and arrays
✅ Built-in validation based on JSON Schema rules
🔄 Two-way data binding with v-model
🎨 Customizable styling
💬 Support for descriptions and custom labels

## Installation

```bash
npm install vue-schema-form
```

## Basic Usage

```vue
<template>
  <JsonSchemaForm
    :schema="schema"
    v-model="formData"
    @submit="handleSubmit"
    @validation-error="handleValidationError"
  />
</template>

<script setup>
import { ref } from 'vue';
import JsonSchemaForm from 'vue-schema-form';

const schema = ref({
  type: "object",
  title: "User Information",
  description: "Please fill in your details",
  properties: {
    name: {
      type: "string",
      title: "Full Name"
    },
    email: {
      type: "string",
      format: "email",
      title: "Email Address"
    },
    age: {
      type: "integer",
      title: "Age",
      minimum: 18
    }
  },
  required: ["name", "email"]
});

const formData = ref({});

const handleSubmit = (data) => {
  console.log('Form submitted:', data);
};

const handleValidationError = (errors) => {
  console.log('Validation errors:', errors);
};
</script>
```

## Props

| Prop | Type | Required | Description |
|------|------|----------|-------------|
| `schema`              | `Object` | *required* | JSON Schema that defines the form structure |
| `modelValue`          | `Object` | `{}`       | Form data (v-model support) |
| `initialData`         | `Object` | `{}`       | Initial data to populate the form
| `submitButtonText`    | `String` | `'Submit'` | Text for the submit button
| `errors`              | `Object` | `{}`       | Custom error messages
| `fieldPrefix`         | `String` | `''`       | Prefix for field IDs (useful for nested forms)
| `level`               | `Number` | `0`        | Nesting level (internal use)

## Events

| Event | Payload | Description |
|-------|---------|-------------|
| submit | Form data object | Emitted when form is submitted |
| validation-error | Error object | Validation errors |
| update:model-value | Form data object | Emitted when form data changes |

## JSON Schema Support
The component supports the following JSON Schema features:

* Types: string, number, integer, boolean, object, array
* Formats: email, date, textarea
* Validations: required, minimum, maximum, minLength, maxLength, pattern
* UI Helpers: title, description, examples, enumNames
* Nested structures: objects and arrays

## Advanced Example
```vue
<template>
  <JsonSchemaForm
    :schema="advancedSchema"
    v-model="formData"
    :initialData="initialData"
    submitButtonText="Save Profile"
    @submit="handleSubmit"
  />
</template>

<script setup>
import { ref } from 'vue';
import JsonSchemaForm from 'vue-schema-form';

const advancedSchema = ref({
  type: "object",
  title: "User Profile",
  properties: {
    personal: {
      type: "object",
      title: "Personal Information",
      properties: {
        name: { type: "string", title: "Name" },
        email: { type: "string", format: "email", title: "Email" }
      },
      required: ["name", "email"]
    },
    preferences: {
      type: "object",
      title: "Preferences",
      properties: {
        theme: {
          type: "string",
          enum: ["light", "dark", "system"],
          enumNames: ["Light Mode", "Dark Mode", "System Default"],
          default: "system"
        },
        notifications: { type: "boolean", default: true }
      }
    },
    skills: {
      type: "array",
      title: "Skills",
      items: {
        type: "object",
        properties: {
          name: { type: "string", title: "Skill Name" },
          level: {
            type: "string",
            enum: ["beginner", "intermediate", "expert"],
            default: "beginner"
          }
        }
      }
    }
  }
});

const initialData = ref({
  personal: {
    name: "Jane Doe"
  }
});

const formData = ref({});

const handleSubmit = (data) => {
  console.log('Profile saved:', data);
};
</script>
```

## Customization
You can customize the appearance of the form by overriding the CSS classes or using scoped styles in your parent component.