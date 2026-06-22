# Contribution 1: Make invited room behavior prettier and sleeker

**Contribution Number:** 1

**Student:** Aneesha Acharya

**Issue:** https://github.com/project-robius/robrix/issues/592

**Status:** Phase II complete

---

## Why I Chose This Issue

I chose issue #592 "Make invited room behavior prettier and sleeker" because 
it aligns with my interest in design and although I do not have experience with Rust, it is something I would like to learn. The issue is labeled "good first issue" and has 
a clear list of what needs to be done.

I'm interested in this because:
1. I left a comment to claim this issue and someone confirmed that it is still open.
2. There are clear instructions on how to run the project and there are resources to learn more about the app.
3. I want to learn how to use Rust.

---

## Understanding the Issue

### Problem Description

When someone invites you to a Matrix room, Robrix shows an InviteScreen with the inviter's details, the room's details, and two buttons: Reject Invite and Join Room. The screen works correctly but it is visually plain and the feedback could be improved to provide a better user experience.

### Expected Behavior

- When the invite is first shown: the Join button shows an enter-room icon, is enabled, and reads "Join Room".
- Once Join is clicked (join in flight): the button shows a loading spinner, is disabled, and reads "Joining...".
- Once JoinRoomResultAction::Joined is received: the button shows a green checkmark, stays disabled, and reads "Joined!".
- Once an invite is rejected / the room is successfully Left: the room tab closes automatically, so the user doesn't have to close it manually.

### Current Behavior

- The Join button uses a static ICON_CHECKMARK in every state.
- The "Joining..." state shows no spinner — only changed button text.
- The "Joined!" state has no green checkmark or color emphasis.
- A successfully rejected (Left) invite shows a "you may close this invite" message but does not close the room tab on its own.

### Affected Components

- src/home/invite_screen.rs — the InviteScreen widget:
  - the live_design! DSL for the accept_button / cancel_button / buttons view,
  - the match self.invite_state { ... } block in draw_walk that sets button text/state,
  - the JoinRoomResultAction / LeaveRoomResultAction event handlers,
  - set_displayed_invite().
- src/shared/icon_button.rs — RobrixPositiveIconButton (button widget type).
- src/shared/bouncing_dots.rs — BouncingDots, the existing loading spinner.
- src/shared/styles.rs — icon constants (ICON_JOIN_ROOM, ICON_CHECKMARK) and COLOR_FG_ACCEPT_GREEN.
- src/app.rs — navigate_to_room, which holds the existing recipe for closing a room/tab (needed for auto-close-on-Left).

---

## Reproduction Process

### Environment Setup (for Windows)

- install Rust and cmake
- run 'cargo run --release' in project folder
- need to  make multiple accounts to test sending invites

### Steps to Reproduce

- Run Robrix (cargo run --release) and log in.
- From a second Matrix account, invite your logged-in account to a room.
- In Robrix, open the invite — the InviteScreen appears.
- Observe the Join Room button before clicking.
- Click Join Room and watch the button during the join.
- Separately, open another invite and click Reject Invite; watch what happens to the tab.


### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** <img width="2549" height="1221" alt="image" src="https://github.com/user-attachments/assets/10f45506-7ed7-4794-a813-36e681eca340" />
<img width="1893" height="974" alt="image" src="https://github.com/user-attachments/assets/fd4968a5-a421-40ea-be7b-01854d42e30e" />

- **My findings:**
- The Join button shows a checkmark icon even before joining (step 4).
- No spinner appears while joining; only the text changes to "Joining..." (step 5).
- After rejecting, the room tab does not auto-close (step 6).
---

## Solution Approach

### Analysis

1. Static icon: the accept_button hard-codes draw_icon.svg: (ICON_CHECKMARK) in the live_design! DSL, and the match self.invite_state block in draw_walk only updates set_text / set_enabled — it never updates the icon. So the icon is fixed regardless of state.
2. No spinner: a Makepad Button can't natively render arbitrary content (like a spinner) in place of its SVG icon, so the "Joining..." state has no loader. The project does have a reusable spinner (BouncingDots), but it isn't used here.
3. No emphasis on success: the "Joined!" arm reuses the same icon and default text color.
4. Tab not auto-closed: the LeaveRoomResultAction::Left handler only sets InviteState::RoomLeft; it doesn't trigger a tab close. The capability now exists (App::navigate_to_room closes a room via DockAction::TabCloseWasPressed + RoomsListUpdate::HideRoom) but isn't wired into the reject flow.

### Proposed Solution

- Make the join button state-driven: swap its icon per InviteState (enter-room icon by default, green checkmark on success), and show a BouncingDots spinner during the in-flight "Joining..." state.
- Per the issue's suggested simpler route, hide the real button and show the spinner in its place during joining, rather than building a new arbitrary-content button widget.
- Wire the LeaveRoomResultAction::Left handler to auto-close the room tab, reusing the same close recipe App::navigate_to_room already uses.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]
When the InviteScreen is shown, the Join button should read "Join Room" with an enter-room icon and be enabled. On click it should become a spinner reading "Joining...". On JoinRoomResultAction::Joined, it should show a green checkmark and "✅ Joined!". When an invite is rejected and the room is successfully Left, the room tab should close on its own.

**Match:** [What similar patterns/solutions exist in the codebase?]
- The enter-room icon already exists: ICON_JOIN_ROOM (resources/icons/join_room.svg), defined in styles.rs and already used in tombstone_footer.rs and add_room.rs. This is a swap, not a new asset.
- State-driven button updates already exist: the match self.invite_state { ... } block in draw_walk already calls set_enabled / set_text per state.
- The spinner already exists: BouncingDots with start_animation(cx) / stop_animation(cx) (it is not called "LoadingSpinner" — that's a description, not a type).
- The green color already exists: COLOR_FG_ACCEPT_GREEN, already used by the invite screen's completion_label.
- The room/tab close recipe already exists in App::navigate_to_room: DockAction::TabCloseWasPressed(LiveId::from_str(room_id)) + RoomsListUpdate::HideRoom { room_id }.

**Plan:** [Step-by-step implementation plan]
1. In the live_design! DSL for accept_button, change the default icon from ICON_CHECKMARK to ICON_JOIN_ROOM.
2. Add a BouncingDots spinner to the buttons view (optionally inside a frame styled like a disabled button), hidden by default.
3. Determine the runtime idiom for changing a RobrixPositiveIconButton's icon — there's no set_icon helper, so confirm whether apply_over is the expected approach. (Main unknown — resolve before coding the icon swaps.)
4. Update the match self.invite_state arms in draw_walk:
  - WaitingOnUserInput: icon ICON_JOIN_ROOM, button visible+enabled, spinner hidden, text "Join Room".
  - WaitingForJoinResult: hide the button, show + start_animation() the spinner, text "Joining...".
  - WaitingForJoinedRoom: stop/hide spinner, show button, icon ICON_CHECKMARK with COLOR_FG_ACCEPT_GREEN, text "✅ Joined!".
  - WaitingForLeaveResult / RoomLeft: ensure the spinner is stopped/hidden.
5. In the LeaveRoomResultAction::Left handler, trigger an auto-close of the invite's room tab using the navigate_to_room recipe. Resolve the ownership boundary first: the dock lives in App, not in InviteScreen, so decide between emitting an action that app.rs handles vs. closing directly. (Design decision to confirm with mentors.)

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]
- [ ] cargo clippy --workspace --all-features is warning-free (CI runs this with RUSTFLAGS: "-D warnings", so warnings fail the build).
- [ ] No new spelling issues (CI runs the typos check via .github/typos.toml).
- [ ] Commit messages are short, imperative, and descriptive (matching git history; the chore: prefix is reserved for automated maintenance commits).
- [ ] PR targets project-robius/robrix:main, references the issue, and includes before/after screenshots or a screen recording (it's a visual change).
- [ ] No unrelated changes; code matches the surrounding style.

**Evaluate:** [How will you verify it works?]
1. cargo build succeeds and cargo clippy --workspace --all-features is clean (matches CI).
2. Manual flow with a test invite:
  - Initial: Join button shows the enter-room icon, is enabled, reads "Join Room".
  - On click: button is replaced by a spinner, reads "Joining...".
  - On success: green checkmark, "✅ Joined!".
  - On reject: the room tab auto-closes.
3. Regression checks: failure cases (JoinRoomResultAction::Failed / LeaveRoomResultAction::Failed) still reset the button to enabled "Join Room"; the confirmation modal path and the Shift-to-skip-confirmation path still work; reject popups still appear.
4. Attach a short screen recording of the join + reject flows to the PR.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
