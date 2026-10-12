# Form state with React Hook Form

The form library owns what the user is typing. One schema owns the rules and the types.
Nothing else validates. Use the schema library the project already has: Zod, Yup or
Valibot, each through its resolver from `@hookform/resolvers`.

## Contents

- [One schema, one type](#one-schema-one-type)
- [Inputs on the web and React Native](#inputs-on-the-web-and-react-native)
- [Rules that depend on runtime data](#rules-that-depend-on-runtime-data)
- [Reading live values](#reading-live-values)
- [Lists of rows](#lists-of-rows)
- [Edit forms seeded from the server](#edit-forms-seeded-from-the-server)
- [Submitting and server errors](#submitting-and-server-errors)

## One schema, one type

**Zod**

```ts
import { zodResolver } from '@hookform/resolvers/zod';
import { useForm } from 'react-hook-form';
import { z } from 'zod';

export const orderFormSchema = z.object({
  customerId: z.string().min(1, 'Choose a customer'),
  quantity: z.number({ error: 'Enter a quantity' }).int().positive('Enter at least 1'),
  notes: z.string().max(500).optional(),
});
export type OrderFormValues = z.infer<typeof orderFormSchema>; // what submit receives

// In the component:
const form = useForm({
  resolver: zodResolver(orderFormSchema),
  defaultValues: { customerId: '', quantity: 1, notes: '' },
});
```

**Yup**

```ts
import { yupResolver } from '@hookform/resolvers/yup';
import * as yup from 'yup';

export const orderFormSchema = yup.object({
  customerId: yup.string().required('Choose a customer'),
  quantity: yup.number().typeError('Enter a quantity').integer().positive('Enter at least 1').required(),
  notes: yup.string().max(500),
});
export type OrderFormValues = yup.InferType<typeof orderFormSchema>;

const form = useForm({
  resolver: yupResolver(orderFormSchema),
  defaultValues: { customerId: '', quantity: 1, notes: '' },
});
```

**Valibot**

```ts
import { valibotResolver } from '@hookform/resolvers/valibot';
import * as v from 'valibot';

export const orderFormSchema = v.object({
  customerId: v.pipe(v.string(), v.minLength(1, 'Choose a customer')),
  quantity: v.pipe(v.number('Enter a quantity'), v.integer(), v.minValue(1, 'Enter at least 1')),
  notes: v.optional(v.pipe(v.string(), v.maxLength(500))),
});
export type OrderFormValues = v.InferOutput<typeof orderFormSchema>;

const form = useForm({
  resolver: valibotResolver(orderFormSchema),
  defaultValues: { customerId: '', quantity: 1, notes: '' },
});
```

The Zod examples use Zod 4. On Zod 3, write `{ invalid_type_error: '…' }` instead of
`{ error: '…' }`, and `z.string().pipe(z.coerce.number())` instead of
`z.coerce.number<string>()`.

Whichever library:

- Never write a separate `interface OrderFormValues`. Infer it from the schema.
- **Leave the type arguments off `useForm`.** The resolver supplies two types: what the
  fields hold (the schema's input) and what `handleSubmit` passes you (its output). They
  differ once the schema coerces, transforms or has defaults (`z.coerce`, `.transform`,
  `.default`), and `useForm<OrderFormValues>` then fails to compile. If you must write them,
  write all three: `useForm<z.input<typeof s>, unknown, z.output<typeof s>>` (Valibot:
  `v.InferInput` and `v.InferOutput`).
- Yup is the exception: `yupResolver` types both sides as `yup.InferType` (the output) and
  hands your submit handler Yup's cast values.
- Give every field a default (`''`, not `undefined`), so inputs start controlled and
  `isDirty` works.
- Never re-check the same rules in the component (`disabled={!customerId || quantity < 1}`).
  Read `formState.isValid`, or let submit show the errors.

## Inputs on the web and React Native

Text inputs hand you strings. On the web, register number fields with `valueAsNumber` for
Zod and Valibot (Yup converts numeric strings itself):

```tsx
<input type="number" {...form.register('quantity', { valueAsNumber: true })} />
```

React Native has no `register`: wrap each input in `Controller`. A `TextInput` always gives a
string, so keep the field a string in the form and let the schema convert it:

```tsx
import { Controller } from 'react-hook-form';
import { Text, TextInput } from 'react-native';

const orderFormSchema = z.object({
  customerId: z.string().min(1, 'Choose a customer'),
  // Holds the typed text; submit receives a number. '' coerces to 0, which .positive()
  // rejects. If 0 is allowed, check for empty text before coercing.
  quantity: z.coerce.number<string>({ error: 'Enter a number' }).int().positive('Enter at least 1'),
});

const form = useForm({
  resolver: zodResolver(orderFormSchema),
  defaultValues: { customerId: '', quantity: '1' },
});

<Controller
  control={form.control}
  name="quantity"
  render={({ field, fieldState }) => (
    <>
      <TextInput
        value={field.value}
        onChangeText={field.onChange}
        onBlur={field.onBlur}
        keyboardType="number-pad"
      />
      {fieldState.error ? <Text>{fieldState.error.message}</Text> : null}
    </>
  )}
/>
```

Valibot does the same with `v.pipe(v.string(), v.toNumber(), v.integer(), v.minValue(1))`.
With Yup the field stays typed as a number: pass `value={String(field.value)}` and Yup
converts the text. Use the same `Controller` on the web for controlled components that
don't take a ref (custom selects, date pickers).

## Rules that depend on runtime data

Build the schema from the data instead of scattering conditions through the JSX.

```ts
export function buildStatusChangeSchema(next: OrderStatus) {
  return z.object({
    status: z.literal(next),
    documentId: requiresDocument(next)
      ? z.string().min(1, 'Upload the required document')
      : z.string().optional(),
  });
}

const schema = useMemo(() => buildStatusChangeSchema(nextStatus), [nextStatus]);
const form = useForm({
  resolver: zodResolver(schema),
  defaultValues: { status: nextStatus, documentId: '' },
});
```

`useForm` reads the latest resolver each time it validates, so a new schema applies from the
next change, blur or submit, and errors already on screen stay until then. If `nextStatus`
can change while the form is open, render the form with `key={nextStatus}` so it starts
fresh. Yup handles the same need with `.when()`; building the schema from data is often
clearer.

## Reading live values

```ts
// Good: re-renders only the component that calls it, and only when `quantity` changes.
const quantity = useWatch({ control: form.control, name: 'quantity' });

// Avoid: re-renders the whole form, and React Compiler skips optimising any component
// that calls it (the `react-hooks/incompatible-library` lint flags it).
const quantity = form.watch('quantity');
```

Call `useWatch` in the small component that shows the value (a running total), not at the
top of the form.

## Lists of rows

```tsx
const { fields, append, remove } = useFieldArray({ control: form.control, name: 'items' });

{fields.map((field, index) => (
  <div key={field.id}>
    <input {...form.register(`items.${index}.sku`)} />
    <button type="button" onClick={() => remove(index)}>Remove</button>
  </div>
))}
<button type="button" onClick={() => append({ sku: '', quantity: 1 })}>Add item</button>
```

- Key rows by `field.id`, never the index.
- `append` takes a complete row, with a default for every field.
- Put list rules in the schema (`z.array(itemSchema).min(1, 'Add an item')`). The message for
  the list itself is at `errors.items?.root?.message` while rows exist, and at
  `errors.items?.message` when the list is empty, so read both.

## Edit forms seeded from the server

The server cache owns the saved order; the form owns the unsaved draft. Mount the form once
the data has loaded, keyed by id, and seed it once:

```tsx
export function EditOrderScreen({ id }: { id: string }) {
  const { data: order } = useOrder(id);
  if (!order) return <OrderFormSkeleton />; // plus the error state
  return <EditOrderForm key={order.id} order={order} />;
}

function EditOrderForm({ order }: { order: Order }) {
  const form = useForm({
    resolver: zodResolver(orderFormSchema),
    defaultValues: toFormValues(order), // read once, on mount
  });
  // ...
}
```

- `defaultValues` is read once, so a background refetch can't overwrite the draft.
- To let a refetch update the fields the user hasn't touched, pass
  `values: toFormValues(order)` with `resetOptions: { keepDirtyValues: true }` instead.
  React Hook Form compares `values` deeply, so a new object each render is fine. Without
  `keepDirtyValues`, every refetch that changes the order wipes what the user typed.
- Never copy the order into `useState` first.
- If the form stays open after saving, call `form.reset(toFormValues(saved))` so it is no
  longer dirty.

## Submitting and server errors

```tsx
const createOrder = useCreateOrder();

const onSubmit = form.handleSubmit(async (values) => {
  try {
    await createOrder.mutateAsync(values); // RTK Query: createOrder(values).unwrap(); SWR: trigger(values)
    onDone();
  } catch (error) {
    // ApiError stands for however your API client reports field errors.
    if (error instanceof ApiError && error.fields.length > 0) {
      for (const { field, message } of error.fields) {
        // `field` must be a form path ('quantity', 'items.0.sku'); map API names that differ.
        form.setError(field as FieldPath<OrderFormValues>, { message });
      }
    } else {
      form.setError('root.server', { message: 'Could not save the order. Try again.' });
    }
  }
});

<form onSubmit={onSubmit} noValidate>
  {/* fields */}
  {form.formState.errors.root?.server ? (
    <p role="alert">{form.formState.errors.root.server.message}</p>
  ) : null}
  <button type="submit" disabled={form.formState.isSubmitting}>Create order</button>
</form>
```

- Await the mutation inside `handleSubmit`. `formState.isSubmitting` stays true until it
  settles, so the button disables itself and a create or payment can't be sent twice.
- Catch errors inside the handler: `handleSubmit` re-throws whatever your handler throws.
- A field error from `setError` clears once that field passes validation again; a `root`
  error clears on the next submit.
- Close the screen or sheet on success; the mutation updates the cache.
- On the web, add `noValidate` to the `<form>` so the browser's own checks (from
  `type="email"`, `required`, `min`) don't block submit before the schema runs. React
  Native has no `<form>`: call `onSubmit()` from the button's `onPress`.
