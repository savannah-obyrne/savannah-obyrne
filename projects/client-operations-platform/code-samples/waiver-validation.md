# Adult Waiver Validation Boundary

**Status:** Adapted excerpt. Unrelated validation branches and implementation details were omitted for focus; the age-routing and signature-consistency behaviour is unchanged.

This focused excerpt shows two rules from a broader waiver schema: minors must use the guardian path, and the entered signature must match the participant name.

```ts
export const waiverAdultInputSchema = z
  .object({
    submissionId: submissionIdSchema,
    firstName: z.string().min(1, "First name is required.").max(100),
    lastName: z.string().min(1, "Last name is required.").max(100),
    email: z
      .string()
      .email("If provided, email must be a valid address.")
      .max(255)
      .optional(),
    phone: z.string().min(7, "Phone number is required.").max(30),
    address: z.string().min(1, "Address is required.").max(300),
    dateOfBirth: ddMmYyyyString,
    emergencyContactName: emergencyContactNameSchema,
    emergencyContactPhone: emergencyContactPhoneSchema,
    signatureName: z.string().min(1, "Signature name is required.").max(200),
    initials: initialsSchema,
    bookingId: z.string().min(1).max(50).optional(),
  })
  .superRefine((data, ctx) => {
    if (!(data.dateOfBirth instanceof Date)) return;
    const age = ageInYears(data.dateOfBirth, currentBusinessDateForAge());
    if (age < 18) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message:
          "Players under 18 must use the guardian form. " +
          "Please have a parent or guardian sign on your behalf.",
        path: ["dateOfBirth"],
      });
    }
    const expected = normaliseName(`${data.firstName} ${data.lastName}`);
    const actual = normaliseName(data.signatureName);
    if (actual !== expected) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: "Signature name must match your first and last name exactly.",
        path: ["signatureName"],
      });
    }
  });
```

**Demonstrates:** validated API input, age-dependent workflow enforcement, and cross-field signature consistency.
