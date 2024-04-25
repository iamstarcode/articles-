---
title: "Validate data in both Server Actions and Client in NextJs using Zod and React-Hook-Forms"
datePublished: Thu Apr 25 2024 18:56:18 GMT+0000 (Coordinated Universal Time)
cuid: clvflubvt000u09l99vazez4e
slug: validate-data-in-both-server-actions-and-client-in-nextjs-using-zod-and-react-hook-forms
tags: reactjs, validation, nextjs, react-hook-form, nextjs-server-actions

---

Server Actions, introduced in Next.js version 14, are a convenient way to create functions that execute **exclusively on the server side** offering several advantages such as security, performance etc. Building robust applications often requires data validation on both the client side (user-interaction) and server-side (backend processing). Validations on the client side give instant feedback to users filling a form, creating a rich user experience, and as they say "never trust the client", it's also as important as validating the client as it's as validating in the backend.

There are several great tools for handling forms. Zod for the creation of **structured definitions** of the expected data format, including data types, constraints, and validation rules. React Hook Form serves as a **powerful library for managing and validating form states in React applications** providing an enhanced user experience.

### Submit Component

Server actions can be invoked using the `action` attribute in an `<form>` element, `useEffect`, third-party libraries, and other form elements like `<button>`.

```typescript
"use client"

import { Loader2 } from "lucide-react"
import { useFormStatus } from "react-dom"

import { Button, ButtonProps } from "@/components/ui/button"

type Props = ButtonProps & {
  pendingText?: string
  text: string
}

export function SubmitButton({ text, pendingText, ...props }: Props) {
  const { pending, action } = useFormStatus()

  const isPending = pending && action === props.formAction

  return (
    <Button {...props} disabled={isPending}>
      {isPending ? <Loader2 className="mr-2 h-4 w-4 animate-spin" /> : null}
      {isPending ? pendingText : text}
    </Button>
  )
}
```

Instead of invoking forms with the form `action` attribute, we can create a simple button component using the React [`useFormStatus`](https://react.dev/reference/react-dom/hooks/useFormStatus) hook to show a pending state while the form is being submitted.

### Server Action and Validation

With zod, we can create the structure of what our data should look like before we can have a valid form. This a simple schema for a to-do creation form.

```typescript
import { z } from "zod"

export const todoSchema = z.object({
  title: z.string().min(5),
  description: z.string().min(5),
})
```

Server Components can use the inline function level or module level `"use server"` directive. To inline a Server Action, add `"use server"` to the top of the function body. When forms are validated on the server and there are errors, we can return serialized objects from our server actions and use React [`useFormState`](https://react.dev/reference/react-dom/hooks/useFormState) to show messages to our user. This changes the signature of our action to receive a new signature to receive a new `prevState` or `initialState` parameter as its first argument

```typescript
"use server"

import { FormState } from "@/types"
import { todoSchema } from "@/utils/schema"

export async function createTodo(prevState: any, formData: FormData) {
  const result = todoSchema.safeParse(Object.fromEntries(formData.entries()))
  if (!result.success) {
    let errorMsgs: string[] = []

    result.error.issues.forEach((issue) => {
      errorMsg.push(issue.path[0] + ": " + issue.message)
    })

    return {
      errors: errorMsgs,
      message: "Error: Please Check Your Input!",
      type: "ValidationError",
    } as FormState
  }
/*
...*/
}
```

### Client-Side Validation

For validation to hit on the client side before submitting the form, we will have the react-hook-form handle form state before submitting. Instead of invoking server action with the `action` attribute of the form, we create a function that handles the validation, triggers validation errors and shows the errors on the form, before submitting.

We will pass the action to the `useFormState` hook, which changes its signature of the action like we have declared earlier to receive a new `prevState` or `initialState` parameter as its first argument,

```typescript
"use client"

import { FormState } from "@/types"
import { todoSchema } from "@/utils/schema"
import { zodResolver } from "@hookform/resolvers/zod"
import { useFormState } from "react-dom"
import { useForm } from "react-hook-form"
import { z } from "zod"

import {
  Card,
  CardContent,
  CardDescription,
  CardFooter,
  CardHeader,
  CardTitle,
} from "@/components/ui/card"
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form"
import { Input } from "@/components/ui/input"
import { createTodo } from "@/app/actions"

import { SubmitButton } from "./SubmitButton"

const initialState: FormState = {
  message: "",
}

export default function Todo() {
  const [state, formAction] = useFormState(createTodo, initialState)

  const form = useForm<z.infer<typeof todoSchema>>({
    resolver: zodResolver(todoSchema),
    mode: "onChange",
    resetOptions: {
      keepValues: false,
    },
    defaultValues: {
      title: "",
      description: "",
    },
  })

  useEffect(() => {
   //handle form state
  }, [state])

  function clientAction(formData: FormData) {
    const result = todoSchema.safeParse(Object.fromEntries(formData.entries()))
    if (!result.success) {
      form.trigger()
    } else {
      formAction(formData)
    }
  }
  return (
    <Form {...form}>
      <form>
        <Card>
          <CardHeader>
            <CardTitle>Create Todo</CardTitle>
            <CardDescription>Form for creating a simple Todo</CardDescription>
          </CardHeader>
          <CardContent className="space-y-4">
            <FormField
              control={form.control}
              name="title"
              render={({ field }) => (
                <FormItem>
                  <div className="flex items-center">
                    <FormLabel>Title</FormLabel>
                  </div>
                  <FormControl>
                    <Input
                      type="text"
                      placeholder="Title"
                      {...field}
                      required
                    />
                  </FormControl>
                  <FormMessage className="text-xs" />
                </FormItem>
              )}
            />
            <FormField
              control={form.control}
              name="description"
              render={({ field }) => (
                <FormItem>
                  <div className="flex items-center">
                    <FormLabel>Description</FormLabel>
                  </div>
                  <FormControl>
                    <Input
                      type="text"
                      placeholder="Description"
                      {...field}
                      required
                    />
                  </FormControl>
                  <FormMessage className="text-xs" />
                </FormItem>
              )}
            />
          </CardContent>
          <CardFooter>
            <SubmitButton
              className="ml-auto"
              formAction={clientAction}
              text="Create Todo"
              pendingText="Creating Todo..."
            />
          </CardFooter>
        </Card>
      </form>
    </Form>
  )
}
```

When the form is submitted the `clientAction` function tries to validate the form, if there are validation errors, the `trigger` function of the form object is called to populate the form using the `FormMessage` component. And if there are any errors server-side the [`useFormState`](https://react.dev/reference/react-dom/hooks/useFormState) sends the serialised object back into `state`, this way making use same schema to validate both the frontend and the backend.