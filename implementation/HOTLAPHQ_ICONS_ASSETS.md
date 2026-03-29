
HotLapHQ Logo Asset List

Primary Master Logo

Asset	Size	Description	Usage
Primary Master Logo	2000 × 2000	Full logo with stripe and colors	Master source file for resizing


⸻

Website Logos

Asset	Size	Description	Usage
Header Logo	1200 × 300	Full HotLapHQ logo with stripe	Website top navigation
Footer Logo	1000 × 300	Full HotLapHQ logo with stripe	Website footer
Email Signature Logo	600 × 200	Full HotLapHQ logo	Email signature


⸻

Caption Image Watermark Logos

Used inside caption images such as 1080 × 1350 social posts.

Asset	Size	Description	Usage
Watermark Black	600 × 150	HOTLAPHQ text only, black	White caption backgrounds
Watermark White	600 × 150	HOTLAPHQ text only, white	Dark caption backgrounds

Notes:
	•	Stripe removed
	•	Monochrome only
	•	Opacity adjusted when placing in design.

⸻

Social Media Profile Icons

Asset	Size	Description	Usage
Social Icon	1000 × 1000	HQ icon	Instagram
Social Icon	1000 × 1000	HQ icon	Facebook
Social Icon	1000 × 1000	HQ icon	Threads
Social Icon	1000 × 1000	HQ icon	Twitter/X
Social Icon	1000 × 1000	HQ icon	BlueSky

All platforms use the same HQ icon.

⸻

Favicon

Asset	Size	Description	Usage
Favicon	512 × 512	HQ icon	Browser favicon

Export PNG → convert later to:

favicon.ico


⸻

Apple Touch Icon

Asset	Size	Description	Usage
Apple Touch Icon	180 × 180	HQ icon	iPhone / iPad home screen


⸻

YouTube Profile Icon

Asset	Size	Description	Usage
YouTube Profile	800 × 800	HQ icon	YouTube channel profile


⸻

Summary

Logo Type	Design
Primary Logo	HOTLAPHQ + stripe
Watermark Logo	HOTLAPHQ (monochrome, no stripe)
Icon Logo	HQ


⸻

Total Logo Files

Category	Files
Master Logo	1
Website Logos	3
Watermark Logos	2
Social Icons	1
Favicon	1
Apple Touch Icon	1
YouTube Icon	1

Total:

10 logo assets

----------

STORAGE

Good — this is how you should organize the brand assets in your repository so that as your project grows you don’t lose track of logo versions. This is the same structure many media projects use.

You can store this inside your repo, for example:

/assets/brand/


⸻

Recommended Folder Structure

assets/
└── brand/
    ├── logo-master/
    │   └── hotlaphq_master_2000x2000.png
    │
    ├── website/
    │   ├── hotlaphq_header_1200x300.png
    │   ├── hotlaphq_footer_1000x300.png
    │   └── hotlaphq_email_signature_600x200.png
    │
    ├── watermark/
    │   ├── hotlaphq_watermark_black_600x150.png
    │   └── hotlaphq_watermark_white_600x150.png
    │
    ├── social/
    │   └── hotlaphq_icon_HQ_1000x1000.png
    │
    ├── youtube/
    │   └── hotlaphq_youtube_icon_800x800.png
    │
    ├── favicon/
    │   ├── favicon_512.png
    │   └── favicon.ico
    │
    └── apple-touch/
        └── apple_touch_icon_180x180.png


⸻

Naming Convention (important)

Use clear names + sizes.

Example:

hotlaphq_header_1200x300.png
hotlaphq_icon_HQ_1000x1000.png
hotlaphq_watermark_black_600x150.png

This prevents confusion later.

⸻

Where each file will be used

Folder	Used in
website	website header / footer
watermark	caption images
social	Instagram, Facebook, Threads, X
youtube	YouTube profile
favicon	browser tab
apple-touch	iPhone home screen
logo-master	original master logo


⸻

Important Rule

Always keep one untouched master file:

hotlaphq_master_2000x2000.png

Never edit this. All other logos are derived from it.

⸻

Later (when the project grows)

You may add:

assets/brand/svg/
assets/brand/press-kit/
assets/brand/social-banners/

But for now the structure above is perfectly sufficient.

⸻

One extra thing I strongly recommend

Create a small file:

assets/brand/brand_guidelines.md

Inside it store things like:

Primary Color: #339967
Font: Josefin Sans
Icon: HQ
Primary Logo: HOTLAPHQ + stripe

This prevents branding inconsistencies later.

⸻