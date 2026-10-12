# Form state with Formik

Formik owns what the user is typing. One schema owns the rules and the types. Nothing else
validates. Formik runs Yup itself (`validationSchema`); with Zod or Valibot, pass a
`validate` function built from the schema.

## Contents

- [Yup](#yup)
- [Zod or Valibot](#zod-or-valibot)
- [Rules](#rules)

## Yup

```ts
import { useFormik } from 'formik';
import * as yup from 'yup';

export const orderFormSchema = yup.object({
  customerId: yup.string().required('Choose a customer'),
  quantity: yup
    .number()
    .typeError('Enter a number')
    .integer()
    .positive('Enter at least 1')
    .required('Enter a quantity'), // Formik turns '' into undefined before Yup sees it
  notes: yup.string().max(500),
});
export type OrderFormValues = yup.InferType<typeof orderFormSchema>;

const formik = useFormik<OrderFormValues>({
  initialValues: { customerId: '', quantity: 1, notes: '' },
  validationSchema: orderFormSchema,
  onSubmit: async (values, { setFieldError, setStatus }) => {
    setStatus(undefined);
    // Formik passes the raw input, not Yup's result: cast applies the conversions.
    const order = orderFormSchema.cast(values);
    try {
      await createOrder.mutateAsync(order);
      onDone();
    } catch (error) {
      // ApiError stands for however your API client reports field errors.
      if (error instanceof ApiError && error.fields.length > 0) {
        for (const { field, message } of error.fields) setFieldError(field, message);
      } else {
        setStatus('Could not save the order. Try again.');
      }
    }
  },
});
```

Show `formik.status` as a form-level message, for example in a `role="alert"` element.

## Zod or Valibot

Formik's `errors` mirror the shape of the values (`{ items: [{ sku: 'Required' }] }`), so
build them with Formik's `setIn`, never as flat `'items.0.sku'` keys.

```ts
import { setIn, useFormik, type FormikErrors } from 'formik';
import { z } from 'zod';

export function zodValidate<Values>(schema: z.ZodType<unknown, Values>) {
  return (values: Values): FormikErrors<Values> => {
    const result = schema.safeParse(values);
    if (result.success) return {};
    let errors: FormikErrors<Values> = {};
    const seen = new Set<string>();
    for (const issue of result.error.issues) {
      // Give cross-field .refine() checks a `path`, or their message has no field to show on.
      const path = issue.path.join('.');
      if (seen.has(path)) continue; // keep the first message for each field
      seen.add(path);
      errors = setIn(errors, path, issue.message);
    }
    return errors;
  };
}

const orderFormSchema = z.object({
  customerId: z.string().min(1, 'Choose a customer'),
  quantity: z.number({ error: 'Enter a quantity' }).int().positive('Enter at least 1'),
});
type OrderFormValues = z.infer<typeof orderFormSchema>;

const formik = useFormik<OrderFormValues>({
  initialValues: { customerId: '', quantity: 1 },
  validate: zodValidate(orderFormSchema),
  onSubmit: async (values) => {
    // parse returns the schema's output; catch and map errors as in the Yup example.
    await createOrder.mutateAsync(orderFormSchema.parse(values));
  },
});
```

For Valibot, the same helper uses `v.safeParse(schema, values)` and `v.getDotPath(issue)`
for the path.

## Rules

- **Types:** infer the values type from the schema; don't declare it twice. If the schema
  transforms values, type Formik with the schema's input (`z.input`) and parse in
  `onSubmit`.
- **No extra validation in the component.** Read `formik.errors` and `formik.isValid`.
  Formik validates every field on each change, so show a field's error only once it is
  touched: `formik.touched.quantity && formik.errors.quantity`. A submit attempt marks every
  field touched.
- **Edit forms:** the server cache owns the saved order; the form owns the unsaved draft.
  Mount the form once the query has data, keyed by id
  (`<EditOrderForm key={order.id} order={order} />`), and build `initialValues` from that
  data, never from a `useState` copy. Leave `enableReinitialize` off: it resets the whole
  form, including what the user typed, whenever `initialValues` changes (for example after a
  refetch on window focus). If existing code needs it, turn off background refetching for
  that query while the form is open.
- **Submitting:** `formik.isSubmitting` stays true while an async `onSubmit` is pending, so
  the button disables itself and nothing is sent twice. Await or return your mutation's
  promise (`mutateAsync` in TanStack Query, `.unwrap()` in RTK Query, `trigger` in SWR).
  Catch errors inside `onSubmit`: Formik only logs a rejected submit as a warning.
- **Server errors:** `setFieldError(field, message)` for field errors (nested paths like
  `'items.0.sku'` work), `setStatus` for a form-level message. Formik's next validation run
  replaces `errors`, so a server field error disappears once the user edits.
- **React Native:** there is no `<form>`. Use `onChangeText={formik.handleChange('quantity')}`,
  `onBlur={formik.handleBlur('quantity')}` and `onPress={() => formik.handleSubmit()}`. A
  `TextInput` gives strings, so number fields hold strings. Yup validates them as numbers
  and the `cast` in `onSubmit` converts them; display them with
  `value={String(formik.values.quantity)}`. With Zod, `z.number()` rejects text, so use
  `z.coerce.number<string>()`, type Formik with `z.input`, and `parse` in `onSubmit`.
- **Rules that depend on runtime data:** build the schema from the data
  (`useMemo(() => buildStatusChangeSchema(nextStatus), [nextStatus])`) and pass it in, or use
  Yup's `.when()`. Formik uses the new schema from its next validation.
- **Lists of rows:** use `<FieldArray name="items">` and its `push` / `remove` helpers. Errors
  for a row are at `errors.items?.[index]`, which is what `setIn` builds.
