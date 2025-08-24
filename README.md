# Virtual-Guitar-using-MATLAB
function virtual_guitar
% VIRTUAL_GUITAR with waveform + spectrogram display
% - Laptop keyboard acts as guitar strings
% - Karplus-Strong plucked string synthesis
% - Waveform + Spectrogram plotted for each note
% - Output works with built-in or Bluetooth speakers

    %configure
    Fs        = 44100;           % sample rate
    baseDur   = 0.80;            % default note duration (seconds)
    volume    = 0.8;             % master volume (0..1)
    brightness= 0.996;           % decay factor for pluck
    polyphony = false;           % allow multiple notes simultaneously?

    % Key → semitone mapping
    keyList = {'a','s','d','f','g','h','j','k','l','semicolon', ...
               'q','w','e','r','t','y','u','i','o','p'};
    midiStart = 52; % E3
    midiNums  = midiStart + (0:numel(keyList)-1);

    key2midi = containers.Map();
    for i = 1:numel(keyList)
        key2midi(keyList{i}) = midiNums(i);
    end

    % UI Figure
    f = figure('Name','Virtual Guitar (MATLAB)','NumberTitle','off', ...
               'Color','w','MenuBar','none','ToolBar','none', ...
               'KeyPressFcn',@onKey,'CloseRequestFcn',@onClose, ...
               'Position', centerFig(1000,600));

    % Layout: 2 plots side by side
    ax1 = subplot(1,2,1,'Parent',f); % waveform
    ax2 = subplot(1,2,2,'Parent',f); % spectrogram
    xlabel(ax1,'Time (s)'); ylabel(ax1,'Amplitude');
    title(ax1,'Waveform','FontWeight','bold');
    title(ax2,'Spectrogram','FontWeight','bold');

    % Info text
    uicontrol('Style','text','Parent',f,'Units','normalized', ...
        'Position',[0.05 0.01 0.9 0.05], ...
        'BackgroundColor','w','FontSize',11,'HorizontalAlignment','center', ...
        'String','Keys: A–; and Q–P | +/- volume | [ ] duration | Esc quit');

    % State
    currentPlayer = [];
    noteDur = baseDur;

    % Key Handler
    function onKey(~,evt)
        k = evt.Key;
        switch k
            case 'escape'
                onClose(); return;
            case {'add','equal'}
                volume = min(1.0, volume+0.05);
                disp(['Volume: ' num2str(volume)]);
                return;
            case {'subtract','minus'}
                volume = max(0.0, volume-0.05);
                disp(['Volume: ' num2str(volume)]);
                return;
            case 'bracketleft'
                noteDur = max(0.10, noteDur-0.05);
                disp(['Duration: ' num2str(noteDur) 's']);
                return;
            case 'bracketright'
                noteDur = min(3.00, noteDur+0.05);
                disp(['Duration: ' num2str(noteDur) 's']);
                return;
        end

        if isKey(key2midi,k)
            m = key2midi(k);
            f0 = midi2freq(m);
            [y,t] = karplusStrong(f0, noteDur, Fs, brightness);
            y = y .* volume;

            % Plot waveform
            cla(ax1);
            plot(ax1,t,y,'b'); grid(ax1,'on');
            xlabel(ax1,'Time (s)'); ylabel(ax1,'Amplitude');
            nm = midiName(m);
            title(ax1, sprintf('Waveform: %s (%.2f Hz)', nm, f0));

            % Plot spectrogram
            cla(ax2);
            spectrogram(y,512,256,512,Fs,'yaxis');
            title(ax2, sprintf('Spectrogram: %s (%.2f Hz)', nm, f0));

            % Play sound
            if polyphony
                p = audioplayer(y, Fs);
                play(p);
            else
                if ~isempty(currentPlayer) && isvalid(currentPlayer)
                    stop(currentPlayer);
                end
                currentPlayer = audioplayer(y, Fs);
                play(currentPlayer);
            end
        end
    end

    function onClose(~,~)
        try
            if ~isempty(currentPlayer) && isvalid(currentPlayer), stop(currentPlayer); end
        catch, end
        delete(f);
    end

    % Helpers 
    function f = midi2freq(m)
        f = 440 * 2.^((m-69)/12);
    end

    function [y,t] = karplusStrong(f0, dur, Fs, alpha)
        N  = max(2, round(Fs/f0));
        buf = 2*rand(1,N)-1;
        L   = round(dur*Fs);
        y   = zeros(1,L);
        for n = 1:L
            idx = mod(n-1, N)+1;
            nxt = alpha * 0.5 * (buf(idx) + buf(mod(idx, N)+1));
            y(n) = buf(idx);
            buf(idx) = nxt;
        end
        y = y / max(1, max(abs(y)));
        t = (0:L-1)/Fs;
    end

    function s = midiName(m)
        names = {'C','C#','D','D#','E','F','F#','G','G#','A','A#','B'};
        pc = mod(m,12)+1;
        oct = floor(m/12)-1;
        s = sprintf('%s%d', names{pc}, oct);
    end

    function p = centerFig(w,h)
        s = get(0,'ScreenSize');
        x = (s(3)-w)/2; y = (s(4)-h)/2;
        p = [x y w h];
    end
end
