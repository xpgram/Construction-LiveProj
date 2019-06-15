# Capstone Stuff

I got to work on a real team for two weeks! That was neat.

It was a two-week sprint. We had daily stand-ups and weekly code-retrospectives.

I didn't see the beginning of the project, and I won't see the end, but the experience was fun. Much better than my day job. We all worked on adding features to a website for a real client using MVC in Visual Studio. Below are some excerpts of what I did.

#### Index:
1. [Chat Popup Resizing](#user-story-fit-this-chat-popup-into-the-whitespace-adjacent-the-dashboard)
2. [SignalR Refactor](#user-story-signalr-refactor)
3. [Dynamic Chat Management Page](#user-story-dynamic-chat-management-page)
4. [Website Guided Tour](#user-story-website-guided-tour)

# Test: Yes

## User Story: Fit this chat-popup into the whitespace adjacent the Dashboard
![Chat Popup](https://github.com/xpgram/Construction-LiveProj/blob/master/My%20Edit%20History/img/Chat%20Popup%20-%20Stretch%201.PNG)

This was my first real assignment. The popup was a little buggy, so in addition to getting it to fit like that, I also cleaned up the CSS so its elements were aligned, and the HTML surrounding its controls so it functions like you would expect. The popup had an issue where it would flicker sometimes when you changed pages or tried to open it. I messed with this thing a lot, as you'll see below, and in the end it was pretty unobtrusive; there when you wanted it, and gone when you didn't.

This cookie saving system, which I still don't think is doing quite what it's supposed to be, I rewrote and it largely solved that flickering problem.

``` Javascript
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

I adjusted it's CSS here to be elastic with a mininum width so that it actually stayed inside the whitespace whenever there was any.

``` CSS
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

This just proves I know what CSS is, though.

## User Story: SignalR Refactor
Another problem that chat window had was that it would refresh the page nearly every time you looked sternly at it. It was also difficult to use for chatting if the conversation were any longer than the element itself. It was using SignalR to push and send messages from one client to the next, but its implementation was piecemeal and hardly coordinated. My next assignment was actually to rewrite the whole thing, and especially to document it because it was dense and confusing.

``` csharp
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

I didn't change much of it structurally, the basic pieces were already there, but I went over every line, broke some blocks off into their own methods for reuseability, and made sure the class was doing what it was supposed to be. I also updated it with features you would expect it to have, like only sending the newest chats on update instead of refreshing the entire discussion history, auto-scrolling to the bottom when a new chat is posted (detailed further in the Javascript below), or only refreshing the discussion history of the client who asked for it. The previous implementation actaully called to refresh every client's window every time someone reloaded a page.

Here's the Javascript half of this solution.

``` Javascript
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
```

This implementation only auto-scrolls to the bottom when the user is already viewing the bottom. It's actually starting to feel like a real chat service.

![Multiple Chat Clients](https://github.com/xpgram/Construction-LiveProj/blob/master/My%20Edit%20History/img/SignalR%20-%20Multi-Client%20Chat%20Populator.PNG)

I remember this assignment seemed huge when I first took it, but that was because it was using ***a library*** and those are scary to me. It also wasn't commented at all, so it wasn't easy to understand what it was doing. Through research and my own tinkering, however, I realized it wasn't truly that big——and that SignalR is really easy to use, actually. I think I'm just chatting to myself at this point, you can go.

## User Story: Dynamic Chat-Management Page
My next assignment was chat-popup-adjacent. I found it easy to continue working in this area because I felt I already knew it so well. Getting to work didn't require any pre-research warm-up.

The chat management page:

![Chat Management Page](https://github.com/xpgram/Construction-LiveProj/blob/master/My%20Edit%20History/img/Manage%20Chat%20-%20Dynamic%20Update%201.PNG)

The page did not have all these fancy buttons and checkboxes. To edit or delete anything, there were two <a> links in those same columns that took you to a separate page to do all your editing. Basic MVC scaffolding stuff. You can imagine, with how many chats there could be, how slow this process might be. I was tasked with making it more dynamic, and keeping it all on one page.
  
I started by adding the features you saw above. And I added this modal.

![Chat Management Page Modal](https://github.com/xpgram/Construction-LiveProj/blob/master/My%20Edit%20History/img/Manage%20Chat%20-%20Dynamic%20Update%202.PNG)

And then I added this Javascript.

``` Javascript
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

This fills in that modal with the message details, pulled from the page, and lets the user (admin) edit them right then and there. It also updates the page dynamically so you feel like you're actually making changes while using it. It doesn't wait for the database to confirm these changes, however, and it wouldn't know about any errors, so technically it lies to people. It feels really good to use, though, and isn't that what America is really about?

It also only lights that delete button up if there are checked boxes. I did forget to disable the button after clicking it, though, so it stays active if you delete anything, even though nothing is checked.

## User Story: Website Guided Tour
The last major thing I did, which felt like the longest in time but certainly wasn't in length, was to add a site demo for new users. Here are some pictures.

![Site Demo 1](https://github.com/xpgram/Construction-LiveProj/blob/master/My%20Edit%20History/img/Dashboard%20-%20Guided%20Tour%201.PNG)

![Site Demo 2](https://github.com/xpgram/Construction-LiveProj/blob/master/My%20Edit%20History/img/Dashboard%20-%20Guided%20Tour%202.PNG)

I used an open-source plugin called Bootstrap-Tour to do this. You can see the script that starts it up below.

``` Javascript
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

The most challenging part of this one was just understanding how the plugin worked. I was nose-deep inside its API documentation for a long time. Most of the problem wasn't even the plugin's fault, it was Bootstrap's most recent security update sabotaging me; I couldn't get any buttons to appear in the popups the script was creating. As much time as it took me to implement and to solve this silly, silly bootstrap issue, it saved me a lot of time writing my own tour script.

I had to selectively modify parts of the plugin, too; in its original, baseline implementation, it was obscuring the navbar while "highlighting" it, which was a z-index issue; and that function just above the tour setup-block styles the highlighted element's backdrop to border the item (like in the second picture) instead of paneling it. Paneling. You know. It looks prettier, and you can actually tell what it is it wants you to see. I call that a victory.

## Final Thoughts?
This is the first time I've ever worked on a team, on a cobbled-together team project. I didn't interact much with many of them, but it was interesting to see how projects like these are run. Our PM narrowed the scope of our user stories enough that it was fairly easy to stay out of each other's way. The only time we did have conflicts (or that I did, anyway) is when I went off scope to fix some little thing that just needed fixing, and even then, it was like twice total. But even so, throughout the two-week sprint, you could see the progress being made by everyone on the team. Again, I think I'm chatting to myself. It was really cool.

I've wanted to be in the tech industry for a long time, and this was a really confirming experience.
