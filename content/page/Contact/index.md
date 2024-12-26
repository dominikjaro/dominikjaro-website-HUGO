---
title: Contact
layout: contact
menu:
    main: 
        weight: -50
        params:
            icon: user

comments: false
---

<form action="https://formspree.io/f/xyzzklge" method="POST">
    <div class="form-group">
        <label for="name">Full Name</label>
        <input type="text" name="name" id="name" required>
    </div>
    <div class="form-group">
        <label for="email">Email</label>
        <input type="email" name="email" id="email" required>
    </div>
    <div class="form-group">
        <label for="message">Message</label>
        <textarea name="message" id="message" rows="5" required></textarea>
    </div>
    <button type="submit">Send Message</button>
</form>

<style>
.form-group {
    margin-bottom: 1rem;
}

label {
    display: block;
    margin-bottom: 0.5rem;
}

input, textarea {
    width: 100%;
    padding: 0.5rem;
    border: 1px solid var(--card-separator-color);
    border-radius: var(--card-border-radius);
    background: var(--card-background);
    color: var(--card-text-color-main);
}

button {
    background: var(--accent-color);
    color: var(--accent-color-text);
    padding: 0.5rem 1rem;
    border: none;
    border-radius: var(--card-border-radius);
    cursor: pointer;
}

button:hover {
    background: var(--accent-color-darker);
}
</style>