Design a simple model of Facebook/Instagram where people can add other people as friends. In addition, people can post messages, and those messages are visible on their friends' pages. The design should be such that it can handle 10M people. There may be, on average, 100 friends each person has. Every day, each person posts around 10 messages on average.

#### Functional Requirements
- Users should be able to upload photos and view the photos they have uploaded.
- Users should be able to follow other users.
- Users can view feeds containing posts from the users they follow.
- User stories automatically expiring at 24 hours.
- Users should be able to like and comment the posts.

#### Nonfunctional Requirements
- 99.9% Availability
- Images should be served in lower qualities for poor internet connections
- 1000 ms latency for images to start loading
