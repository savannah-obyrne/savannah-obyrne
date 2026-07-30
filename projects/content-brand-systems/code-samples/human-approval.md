# Human Approval and Immutable Revision State

**Status:** Adapted excerpt. A private reviewer identifier and internal labels were replaced with generic public-safe terms; control flow is unchanged.

The service refuses to change revisions that are no longer drafts, requires an explicit override for known warnings, supersedes rather than deletes an earlier approved version, and records the decision as an event.

```py
def _require_draft(revision: Revision) -> None:
    if revision.status != REVISION_DRAFT:
        raise RevisionError(
            f"Revision is '{revision.status}', not a pending draft. "
            "Approved, rejected, or superseded revisions cannot change status again."
        )


def approve(
    db: Session,
    project: Project,
    revision: Revision,
    *,
    reason: str = "",
    override: bool = False,
) -> Approval:
    _require_draft(revision)
    warning = warning_for(revision)
    if warning and not override:
        raise ApprovalWarning(warning)

    previous = latest_approved_revision(
        db, project.id, revision.artifact_kind
    )
    if previous and previous.id != revision.id:
        previous.status = REVISION_SUPERSEDED

    revision.status = REVISION_APPROVED
    approval = Approval(
        project_id=project.id,
        revision_id=revision.id,
        artifact_kind=revision.artifact_kind,
        decision=DECISION_APPROVED,
        reason=reason,
        decided_by=LOCAL_REVIEWER,
    )
    db.add(approval)
    db.add(Event(
        project_id=project.id,
        kind="approval",
        summary=f"Revision {revision.revision_number} approved",
        detail=reason,
    ))

    if warning and override:
        db.add(Event(
            project_id=project.id,
            kind="override",
            summary="Warning explicitly overridden",
            detail=warning,
        ))

    db.commit()
    db.refresh(approval)
    return approval
```

**Demonstrates:** state-transition enforcement, retained revision history, explicit warning overrides, and auditable human decisions.
