import Phaser from 'phaser';
import dragonBones from 'dragonbones-phaser';
import biped_ske_json from './biped_ske.json';
import biped_tex_json from './biped_tex.json';
import biped_tex_png from './biped_tex.png';

export const game = new Phaser.Game({
    type: Phaser.WEBGL,
    width: 400,
    height: 300,
    parent: 'phaser-example',
    plugins: {
        global: [{
            key: "DragonBonesPlugin",
            plugin: dragonBones.phaser.plugin.DragonBonesPlugin,
            start: true
        }],
    },
    scene: [ class extends Phaser.Scene {
        constructor() {
            super({
                key:'Game',
                // physics: {
                //     arcade: { debug: false },
                // }
            });
        }
        preload() {
            super.preload();
            // this.input.mouse.disableContextMenu();
            this.cameras.main.setBackgroundColor('#FFFFFF');
            this.load.dragonbone(
                "kenny",
                biped_tex_png,
                biped_tex_json,
                biped_ske_json,
            );
        }
        create() {
            super.create();
            const armatureDisplay = this.add.armature("kenny");
            armatureDisplay.animation.play("idle_south");
            armatureDisplay.x = this.cameras.main.centerX;
            armatureDisplay.y = this.cameras.main.centerY + 200;

        }
    }],
});
