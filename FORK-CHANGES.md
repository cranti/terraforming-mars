
# Changes from the opensource version

## ⬤ Mobile / bottom navigation

The game view has a fixed bottom nav bar with shortcut buttons that scroll to key sections of the page:

| Button      | Scrolls to        | Contents |
|-------------|-------------------|----------|
| Board       | `section-board`   | Mars board, Turmoil*, Moon Board*, Planetary Tracks*, Milestones & Awards, Colonies* |
| All Players | `section-players` | Players overview |
| Log         | `section-log`     | Game log |
| Play        | `section-play`    | Actions, Drafted Cards*, Cards In Hand*, Played Cards, Self-replicating Robots*, Underground Tokens* |

*\* = shown only when relevant to the current game options/state*

The **Play** button turns red when it's your turn (i.e. you have a pending action)

## Board scaled up

The Mars board is rendered at 115% size (`transform: scale(1.15)`).

## Larger game log

The log panel height is doubled (240px → 480px). The log is pinned to the bottom on load, with a deferred scroll via `IntersectionObserver` to handle cases where the panel isn't yet visible.

## Stolen resource highlighting

Log messages containing "stole" are highlighted with a red background stripe (`.log-steal`).

## Dim unplayable cards

Cards in hand that cannot currently be played are dimmed. The server computes the set of playable cards and passes an `enabled` flag per card in the `PlayerViewModel`.

## Touch drag-and-drop for card reordering

`SortableCards` now supports touch-based drag-and-drop on mobile:
- A ghost clone of the dragged card follows the finger
- Cards show a left/right highlight border indicating where the card will be inserted
- Drag is initiated after 8px of movement to avoid interfering with taps

## ConfirmDialog fix

`ConfirmDialog` is wrapped in a Vue `<Teleport to="body">` so the `<dialog>` element is not clipped by the fixed bottom navigation bar.
