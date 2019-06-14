## Chat Popup Cookie Saving System

```javascript
// Called by the chat layout when it is created.
// Assigns event methods which create/remove a cookie remembering the chat box's collapsed state.
// Also recreates the previous session's state based on present chat box cookies.
function keepChatState(nameOfDiv, nameOfHeader) {
    // When the chatbox body is shown, remove the cookie.
    $("#" + nameOfDiv).on('shown.bs.collapse', function () {
        $.removeCookie(nameOfDiv);
    });
    // When the chatbox body is collapsed, save a cookie whose presence auto-hides it next session.
    $("#" + nameOfDiv).on('hidden.bs.collapse', function () {
        $.cookie(nameOfDiv, "true"); // this is a cookie.
    });

    // Confirm the existence of session cookies
    var showDiv = $.cookie(nameOfDiv);
    var chatFormDiv = $.cookie("chatForm");

    // If the chatbox body cookie is not present, uncollapse the chat dialog instantly.
    if (showDiv == null) {
        $("#" + nameOfDiv).addClass("in");                     // The div to show - 'in' means 'display uncollapsed' to something
        $("#" + nameOfHeader).removeClass("collapsed");        // The header to stylize as expanded - 'collapsed' is a flag
    }
    // If chatform cookie is not present, auto-reveal the chat box instantly.
    if (chatFormDiv == null) {
        $('#chatForm').show();
    }
};

function closeChat() {
    $.cookie("chatForm", "true");   // Adds cookie to maintain state through separate sessions
    $("#chatForm").fadeOut(150);    // Makes chat invisible
};

function openChat() {
    $.removeCookie("chatForm");     // Removes cookie that is added when chat is opened - returns state to default behavior
    $("#chatForm").fadeIn(150);     // Makes chat visible
};

// Takes a given string and sanitizes it. Ex: '>' characters render as "&gt;"
// This is extremely important in preventing script injection attacks.
function htmlEncode(value) {
    var encodedValue = $('<div />').text(value).html();
    return encodedValue;
}
```

## Chat Popup => Elastic Values

```css
.chat-popup {
    width: 24%;
    min-width: 300px;
}

textarea {
    vertical-align: bottom;
    width: 74%;
    height: 50px;
    resize: none;
}

.modal-header {
    background-color: #214565;
    border-radius: 5px;
}

.ei-chatbox-header {
    padding-top: 5px;
    padding-bottom: 5px;
}

.ei-chatbox-body {
    padding-left: 3%;
    padding-right: 3%;
}
```

##  Chat Management - Edit and Delete - HTML

```html
@using ConstructionNew.Models
@model IEnumerable<ConstructionNew.Models.ChatMessage>
@{
    ViewBag.Title = "Index";
}

<h2>Manage Chat</h2>

<div>
    <table class="table">
        <tr>
            <th><h4><strong>Timestamp</strong></h4></th>

            <th><h4><strong>Sender</strong></h4></th>

            <th><h4><strong>Messages</strong></h4></th>

            <th></th> <!-- Blank header column (Edit Buttons) -->

            <th><button type="submit" class="btn btn-danger DeleteButton" disabled>Delete</button></th>
        </tr>

        @foreach (var item in Model)
        {
            <tr>
                <td>
                    @Html.DisplayFor(modelItem => item.Date)
                </td>
                <td class="messageSender">
                    @Html.DisplayFor(modelItem => item.Sender)
                </td>
                <td class="messageContent">
                    @Html.DisplayFor(modelItem => item.Message)
                </td>
                <td>
                    <button type="button" class="btn btn-sm btn-primary EditButton" value="@item.ChatMessageId">Edit</button>
                </td>
                <td>
                    <input type="checkbox" class="checkbox DeleteCheckbox" value="@item.ChatMessageId" />
                </td>
            </tr>
        }

    </table>
</div>

<!--EditForm modal-->
<div id="EditMessageModal" class="modal fade" tabindex="-1" role="dialog">
    <div class="modal-dialog" role="document">
        <div class="modal-content">
            <div class="modal-header text-center" style="background-color: lightgray">
                <h4 class="modal-title">Edit Message</h4>
            </div>
            <div class="modal-body text-center">

                <div class="form-horizontal">
                    <input type="hidden" id="EditForm-MessageID" value="" />

                    <div class="form-group">
                        <!-- Sender (Uneditable) -->
                        <label class="control-label col-sm-2">Sender</label>
                        <div class="col-sm-10">
                            <input type="text" id="EditForm-SenderDisplay" class="form-control" value="" disabled />
                        </div>
                        <br />
                        <!-- Message -->
                        <label class="control-label col-sm-2">Message</label>
                        <div class="col-sm-10">
                            <input type="text" id="EditForm-Message" class="form-control" name="Message" value="" />
                            @*@Html.ValidationMessageFor(model => model.Message, "", new { @class = "text-danger" })*@
                            @*This enabled an error message in some case I couldn't identify.*@
                            @*I don't know how to keep that working on ~this~ page, however.*@
                        </div>
                    </div>
                    <div class="form-group">
                        <div class="col-sm-offset-2 col-sm-10">
                            <button type="button" class="btn btn-success" id="EditForm-SaveButton">Save</button>
                            <button type="button" class="btn btn-danger" id="EditForm-CancelButton">Cancel</button>
                        </div>
                    </div>
                </div>

            </div>
        </div>
    </div>
</div>

<div>
    @Html.ActionLink("Back to Dashboard", "Index", "Dashboard")
</div>

<script>
    InitiateChatManagementPage();
</script>
```

##  Chat Management - Edit and Delete - Javascript

```javascript
// Sets up the Manage Chat view's Javascript. Only runs on that page.
function InitiateChatManagementPage() {
    // Function which opens the edit message dialog when an edit button is clicked.
    $('.EditButton').click(function () {
        // Get the chat's ID from the button
        let messageId = $(this).val();

        // Get the important display data
        let tableRow = $(this).parents("tr");
        let messageSender = $(tableRow).children(".messageSender").text().trim();
        let messageContent = $(tableRow).children(".messageContent").text().trim();

        // Fill that data into the edit form
        $('#EditForm-MessageID').val(messageId);
        $('#EditForm-SenderDisplay').val(messageSender);
        $('#EditForm-Message').val(messageContent);

        // Reveal the edit form.
        $('#EditMessageModal').modal();
    });

    // Function which enables the delete-messages button only when at least one message is
    // selected for delete. 
    $('.DeleteCheckbox').click(function () {
        let selected = $('input:checked');
        let disable = (selected.length == 0);
        $('.DeleteButton').prop('disabled', disable);
    });

    // Function which, when the delete button is clicked, 
    // uses the ChatHub server to send a delete request for the selected chat items.
    $('.DeleteButton').click(function () {
        let chat = $.connection.chatHub;

        // Gather a list of all IDs for selected messages as strings
        let messageIds = [];
        $('input:checked').each(function () {
            messageIds.push($(this).val());
        });

        // Open a confirmation modal?
        //   If so, the below stuff needs to be put in a DeleteConfirmButton.click function.

        // Issue the delete request
        chat.server.deleteMessages(messageIds);

        // Update the manage chats view with the items we've deleted.
        // This is technically a lie; the DB probably hasn't finished updating by this point.
        messageIds.forEach(function (id) {
            $('button[value="' + id + '"]').parents("tr").remove();
        });
    });

    // Function which focuses the edit form's message input box when shown.
    $('#EditMessageModal').on('shown.bs.modal', function () {
        $('#EditForm-Message').focus();
    });

    // Function which saves the message's new contents when the edit form's save button is clicked.
    $('#EditForm-SaveButton').on('click', function () {
        // Gather important details.
        let chat = $.connection.chatHub;
        let messageId = $('#EditForm-MessageID').val();
        let messageContent = $('#EditForm-Message').val();

        // Request the server to make the change.
        chat.server.editMessage(messageId, messageContent);

        // Update the view with our changes.
        // This finds the edit button with the right ID, then finds the adjacent table column for message text.
        // This is also technically a lie; the DB probably hasn't finished updating by this point.
        $('button[value="' + messageId + '"]').parents("tr").children(".messageContent").text(messageContent);

        // Hide the edit form.
        $('#EditMessageModal').modal('hide');
    });

    // Function which 'clicks' the edit form's save button when enter is pressed.
    $('#EditForm-Message').on('keypress', function (event) {
        const EnterKeyCode = 13;
        if (event.which == EnterKeyCode && !event.shiftKey)
            $('#EditForm-SaveButton').click();
    });

    // Function which hides the edit form modal when the cancel button is clicked.
    $('#EditForm-CancelButton').on('click', function () {
        $('#EditMessageModal').modal('hide');
    });
};
```

## SignalR Refactor - ChatHub C#

```csharp
namespace SignalRChat
{
    public class ChatHub : Hub //Hub base class provides methods that communicates with signalR connections
    {
        // Link to the website's DB, used for the ChatMessages table.
        ApplicationDbContext db = new ApplicationDbContext();


        // Submits a new chat message to the DB and updates all clients with it.
        public void Send(string name, string message) //This send method is called from Ajax
        {
            ChatMessage chat = new ChatMessage();

            // Fill in the chat model's basic details.
            chat.Message = message;
            chat.ChatMessageId = Guid.NewGuid();
            chat.Date = DateTime.Now;

            // Fill in the chat model's Sender
            string currentUserId = HttpContext.Current.User.Identity.GetUserId();              // Get ID of current user
            ApplicationUser currentUser = db.Users.FirstOrDefault(x => x.Id == currentUserId); // Get name of current user using ID
            chat.Sender = currentUser.FName;                                                   // Assign 'Sender' (a string) the value of [FirstName] [LastInitial]
            chat.Sender += (currentUser.LName != null) ? (" " + currentUser.LName[0]) : "";    // And ignore that last name if the user does not have one.

            // Add the chat item to DB and save changes
            db.ChatMessages.Add(chat);
            db.SaveChanges();

            // Update all clients with the new chat
            PostNewMessageToClients(chat);

            // Force the calling client to look at the newest chats
            Clients.Caller.scrollToBottom();
        }

        // Grabs the entire chat discussion history from the DB and posts it to the user calling for it.
        // Does not affect any other ChatHub clients.
        public void GetMessages()
        {   
            // Clears the calling-client's chat of all content.
            Clients.Caller.refreshChat();

            // Gather and post messages to the chat dialog from the DB
            var messages = db.ChatMessages.ToList();  
            foreach (ChatMessage chatMessage in messages.OrderBy(d => d.Date))
            {
                PostNewMessageToCallingClient(chatMessage);
            }

            // Forces the calling-client to view the newest chats at the bottom of the chat-window element.
            Clients.Caller.scrollToBottom();

            // What to do with this... I feel like this should be somewhere else.
            // I think the server ought to have a periodic update() or clean() method, that's where this should go.
            HostingEnvironment.QueueBackgroundWorkItem(cancellationToken => AutoDeleteChatMessagesAsync(cancellationToken));
        }

        // Edits a message in the DB, locating it by ID, then pushes this change to no clients.
        // Returns true when (if?) the change was successful.
        public void EditMessage(string messageId, string messageContent)
        {
            Guid guid;
            if (Guid.TryParse(messageId, out guid))
            {
                // Find the chat message and update its contents.
                var message = db.ChatMessages.Find(guid);
                message.Message = messageContent;

                // Inform the DB of changes and save them.
                db.Entry(message).State = EntityState.Modified;
                db.SaveChanges();
            }

            // Push this change to all clients
            //Clients.All.refreshChat();    // Happens too quickly, I think. Blanks the chat window instead of updating anything.
        }

        // Deletes a message from the DB by its ID, then pushes that update to nobody.
        // This method creates a one-item array and passes control over to the mains.
        public void DeleteMessage(string chatMessageStringId)
        {
            DeleteMessages(new string[] { chatMessageStringId });
        }

        // Deletes a list of messages from the DB by their IDs, then pushes that update to nobody.
        // The way it does this pushing is a little clunky, though, bear in mind.
        public void DeleteMessages(string[] chatMessageStringIds)
        {
            Guid guid = Guid.Empty;
            var chatMessageGuids = new List<Guid>();

            // Convert all string IDs passed to Guid's (keeps only the valid ones; invalids are ignored!)
            chatMessageGuids = (from strId in chatMessageStringIds
                                where Guid.TryParse(strId, out guid)
                                select guid).ToList();

            // Remove all identified chats from the DB
            foreach (Guid chatId in chatMessageGuids)
                db.ChatMessages.Remove(db.ChatMessages.Find(chatId));

            // Finalize changes
            db.SaveChanges();

            // Push to all clients
            //Clients.All.refreshChat();    // Happens too quickly, I think. Blanks the chat window instead of updating anything.
        }

        // This only needs to be called once a day. Currently, it's sitting in the GetMessages() method above. That means it's called a lot.
        // Cleans the DB of chat messages which are too old.
        private async Task AutoDeleteChatMessagesAsync(CancellationToken cancellationToken)
        {
            const int DaysToKeepChats = 7;

            await Task.Run(() => {
                // Build a list of chats that are too old to keep.
                List<ChatMessage> chatsToDelete = db.ChatMessages.Where(x => DbFunctions.DiffDays(x.Date, DateTime.Now) > DaysToKeepChats).ToList();

                // If this list isn't empty, delete each item from its corresponding row in the DB.
                if (chatsToDelete != null)
                {
                    foreach (var chat in chatsToDelete)
                    {
                        var chatMessage = db.ChatMessages.Find(chat.ChatMessageId);
                        db.ChatMessages.Remove(chatMessage);
                        db.SaveChanges();
                    }
                }
            }, cancellationToken);
        }

        // Posts a new chat message to all clients of the chat hub.
        private void PostNewMessageToClients(ChatMessage chatMessage)
        {
            string date, sender, message;
            FormatNewMessage(chatMessage, out date, out sender, out message);
            Clients.All.addNewMessageToPage(date, sender, message);
        }

        // Posts a new chat message to the client who invoked the server.
        // Currently, this is primarily used by the GetAllMessages() method.
        private void PostNewMessageToCallingClient(ChatMessage chatMessage)
        {
            string date, sender, message;
            FormatNewMessage(chatMessage, out date, out sender, out message);
            Clients.Caller.addNewMessageToPage(date, sender, message);
        }

        // Formats the input fields for a Javascript method call.
        // Currently, this only does anything to the date field.
        private void FormatNewMessage(ChatMessage chatMessage, out string date, out string sender, out string message)
        {
            date = $"{chatMessage.Date.ToString("h:mm tt")}";
            sender = $"{chatMessage.Sender}";
            message = $"{chatMessage.Message}";
        }
    }
}
```

## SignalR Refactor - Javascript

```javascript
<script src="~/Scripts/jquery.signalR-2.4.1.min.js"></script>
<!--Reference the autogenerated SignalR hub script.-->
<script src="~/signalr/hubs"></script>
<!--Script to keep chat message state-->
<script language="javascript" type="text/javascript">
    // JQuery for creating a ChatBox connection 
    // Creates a connection to the chathub server, attaches some Javascript methods to it, and does some light setup.
    $(function () {
        const EnterKeyCode = 13;

        // Reference the auto-generated proxy for the hub.
        let chat = $.connection.chatHub;

        // Function to clear a client's chat dialog of all messages
        chat.client.refreshChat = function () {
            $("#discussion > li").remove();
        };

        // Function to scroll the chatbox's view to the bottom of the message list.
        chat.client.scrollToBottom = function () {
            let element = document.getElementById("discussion");
            element.scrollTop = element.scrollHeight - element.clientHeight;
        }

        // Function returns true if the chatbox's view is currently at the bottom of the chat window (viewing the newest chats).
        chat.client.viewingNewestChats = function () {
            let element = document.getElementById("discussion");
            let value = element.scrollHeight - element.scrollTop - element.clientHeight;
            return Math.floor(value) === 0;  // floor() is important because Chrome has some weirdness with element.scrollTop
        }

        // Function to append one new message to the bottom of the chat box's message list.
        chat.client.addNewMessageToPage = function (time, name, message) {
            let trackNewChats = chat.client.viewingNewestChats();

            // Targets the unordered list of chats and appends a new one.
            $('#discussion').append('<li><strong>' + htmlEncode(name)       // htmlEncode() prevents script-injection attacks
                + '</strong>&nbsp;&nbsp;' + htmlEncode(time) + "</li><li>" + htmlEncode(message) + '</li>');

            // If the user is viewing the most recent chats, keep it that way.
            if (trackNewChats) {
                chat.client.scrollToBottom();
            }
        };

        // When the 'Send' button is clicked: submit new chat and clear the text input box.
        $('#sendmessage').click(function () {
            // Get the user's message and clear the input box.
            let name = $('#displayname').val();
            let message = $('#messageBox').val();
            $('#messageBox').val('');

            // If the submitted chat message is only whitespace, discard it - quit this function early.
            if (message.trim() === "")
                return;

            // Ask the chathub server to submit the message and refresh the view.
            chat.server.send(name, message);

            // Focus the text box; let the user immediately start typing a new message.
            $('#messageBox').focus();
        });

        // When Enter is pressed inside the chatbox's textbox, 'click' the 'Send' button.
        $('#messageBox').keypress(function (event) {
            if (event.which == EnterKeyCode && !event.shiftKey) {
                $('#sendmessage').click();
                event.preventDefault();     // Prevents the enter key from typing a character.
            }
        });

        // Start the connection. This is where anything that needs to happen on initialization goes.
        // Leave this at the bottom of this function; chat.server.getMessages() depends on the Javascript functions defined above.
        // Unless leaving it at the bottom isn't important.
        $.connection.hub.start().done(function () {
            // Get all discussion messages.
            chat.server.getMessages();
        });
    });
    $(document).ready(function(){
        keepChatState("chatbody", "close")
    });
</script>
```

## Chat Popup HTML

```html
<!-- Modal -->
<div class="chat-popup" id="chatForm">
    <!-- Modal content-->
    <div class="modal-content" method="post">
        <div class="modal-header ei-chatbox-header">
            <button type="button" id="close" onclick="closeChat()" class="close">&times;</button>
            <button type="button" id="minimize" class="close" data-toggle="collapse" data-target="#chatbody">&minus;</button>
            @if (User.IsInRole("Admin"))
            {
                @Html.ActionLink("MANAGE", "Index", "ChatMessages", null, new { @class = "manageChat" })
            }
            <h4 class="chatName">@User.Identity.Name</h4>
        </div>
        <form>
            <!-- Wrapper for body content (without any padding) allows smooth animation -->
            <div id="chatbody" class="collapse">
                <div class="modal-body ei-chatbox-body">
                    <ul id="discussion"></ul>
                    <textarea placeholder="Type message..." name="msg" id="messageBox" required></textarea>
                    <input type="hidden" id="displayname" value="@User.Identity.Name" />
                    <button type="button" class="btn ei-message-btn" id="sendmessage">Send</button>
                </div>
            </div>
        </form>
    </div>
</div>
```

## New User Demo - Buttons

```html
<li id="tourButton"><a role="button" hidden>Site Tour</a></li>
<li id="chatButton"><a role="button" onclick="openChat()">Chat</a></li>
```

## New User Demo - Javascript

```javascript
<!-- bootstrap-tour script for new user onboarding-demo. Only needed in Employee view. -->
@Scripts.Render("~/bundles/bootstrap-tour")
<script language="javascript" type="text/javascript">

    $(document).ready(function () {

        // Reveal the Demo button
        $('#tourButton').show();

        // Function which changes the background style behind highlighted tour elements. (Used by some steps in the tour)
        function borderedTourItem(tour) {
            $('.tour-step-background').css('background', 'rgba(255,255,255,.15)');
            $('.tour-step-background').css('border', '2px solid #EDD');
        }

        // Setup the user onboarding tour
        var tour = new Tour({
            // Steps are shown in the order they're added. See Bootstrap-Tour documentation for how to customize.
            steps: [
                {
                    // Large schedule window
                    element: '#schedulePanel',
                    title: 'Your Entire Work Schedule',
                    content: "This window lists every scheduled job you've been assigned to. It even details when you aren't working.",
                    placement: 'top'
                },
                {
                    // Large company news window
                    element: '#newsPanel',
                    title: 'Stay Updated',
                    content: 'This window contains important announcements and company news. Check it often!',
                    placement: 'top'
                },
                {
                    // Targets the chat-window. This only shows if the chat-window is open - i.e. the element isn't 'hidden'.
                    element: '#chatForm',
                    title: 'Need To Ask A Question?',
                    content: 'Use the chat window to communicate with your colleagues.',
                    placement: 'left'
                },
                {
                    // Targets the schedule drop-down menu in the navbar
                    element: '#schedule',
                    title: 'Everyone\'s Schedule',
                    content: 'If you need to know when someone else is working, a link to the company schedule can be found here.',
                    placement: 'bottom',
                    onShown: borderedTourItem
                },
                {
                    // Targets the company drop-down menu in the navbar
                    element: '#company',
                    title: 'Missed An Announcement?',
                    content: 'A link to all the company\'s announcements can be found here.',
                    placement: 'bottom',
                    onShown: borderedTourItem
                },
                {
                    // Targets the chat-button in the navbar.
                    element: '#chatButton',
                    title: 'Open The Chat Popup',
                    content: 'If you\'ve closed the chat popup, you can open it again here.',
                    placement: 'bottom',
                    onShown: borderedTourItem
                },
                {
                    // Targets the MyAccount link in the navbar
                    element: '#userAccount',
                    title: 'Account Details',
                    content: 'You can change your password and personal details here.',
                    placement: 'bottom',
                    onShown: borderedTourItem
                }

                // Do not put hideable elements as the last step - unless the user can't hide them during the tour.
                // When they're hidden, Bootstrap-Tour doesn't know how to 'skip' them because there's nothing to skip to.
                // It breaks the tour.

                // Note: The entire navbar is hideable in sub-768px views. Resizing the window to very, very small breaks the tour.
            ],

            // Bootstrap-Tour Settings:
            backdrop: true,     // Use a black, transparent background so elements stand out
            debug: false
        });

        tour.init();    // Important step - must call before tour.start()
        tour.end();     // Ensures tour.ended() returns true and allows tour.restart() to work, which is preferred.

        // Add a listener to the Demo button in the navbar to start the tour - only works if the tour isn't active.
        $('#tourButton').on('click', function () {
            if (tour.ended()) {
                tour.restart();     // using restart() ensures the user can run the tour as many times as they want.
            }
        });
    });

</script>
```

